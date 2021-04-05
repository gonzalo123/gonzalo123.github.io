---
layout: post
title: "Inject dependencies via PhpDoc"
date: "2012-04-09"
tags: 
  - "pdo"
  - "php"
  - "di"
  - "pdo"
  - "php-class"
  - "phpdoc"
  - "programming"
---

Last month I attended to [Codemotion](http://www.codemotion.es/) conference. I was listening to a talk about Java and I saw the “@inject” decorator. I must admit I switched off my mind from the conference and I started to take notes in my notebook. The idea is to implement something similar in PHP. It's a pity we don't have real decorators in PHP. I really [miss](http://gonzalo123.wordpress.com/2009/12/09/things-i-miss-in-php-function-decorators/ "Things I miss in PHP: Function decorators") them. We need to use [PhpDoc](http://gonzalo123.wordpress.com/2011/02/01/function-decorators-in-php-with-phpdoc-and-annotations/ "Function decorators in PHP with PHPDoc and Annotations"). It's not the same than real decorators in other programming languages. That's my prototype. Let's go.

Imagine this simple class: 

```php
class User
{
    private $db;
 
    public function getInfo($uid)
    {
        $sql = "select * from users where uid=:UID";
        $stmp = $this->db->prepare($sql);
        $stmp->execute(array('UID' => $uid));
        return $stmp->fetchAll();
    }
 
    public function getDb()
    {
        return $this->db;
    }
}
```

It doesn't work. Private property $db must be an instance of PDO object. We can solve it with dependency injection:

```php
class User
{
    private $db;
    public function __construct(PDO $db)
    {
        $this->db = $db;
    }
 
    public function getInfo($uid)
    {
        $sql = "select * from users where uid=:UID";
        $stmp = $this->db->prepare($sql);
        $stmp->execute(array('UID' => $uid));
        return $stmp->fetchAll();
    }
}
```

Now it works but we are going to create a simple PDO Wrapper to obtain the PDO connection. 

```php
class DbConf
{
    const DB1 = 'db1';
 
    private static $conf = array(
        self::DB1 => array(
            'dsn'  => 'pgsql:dbname=db;host=localhost',
            'user' => 'gonzalo',
            'pass' => 'password',
        )
    );
 
    public static function getConf($key)
    {
        return self::$conf[$key];
    }
}
 
class Db extends PDO
{
    private static $dbInstances = array();
 
    /**
     * @static
     * @param string $key
     * @return PDO
     */
    static function getDb($key)
    {
        if (!isset($dbInstances[$key])) {
            $dbConf = DbConf::getConf($key);
            $dbInstances[$key] = new PDO($dbConf['dsn'], $dbConf['user'], $dbConf['pass']);
        }
        return $dbInstances[$key];
    }
}
```

I like to use this kind of classes because I normally work with different databases and I need to use different connection depending on the SQL. It helps me to mock the database connection within the different environments (development, production, QA). Now We can use our simple class:

```php
$user = new User(Db::getDb(DbConf::DB1));
print_r($user->getInfo(4));
```

The idea is to change the class into something like this:

```php
class User extends DocInject
{
    /**
     * @inject Db::getDb(DbConf::DB1)
     * @var PDO
     */
    private $db;
 
    public function getInfo($uid)
    {
        $sql = "select * from users where uid=:UID";
        $stmp = $this->db->prepare($sql);
        $stmp->execute(array('UID' => $uid));
        return $stmp->fetchAll();
    }
}
```

Now we are going to inject the PDO connection to $db private property in the constructor:

```php
class DocInject
{
    public function __construct()
    {
        $reflection = new ReflectionClass($this);
        foreach ($reflection->getProperties() as $property) {
            /** @var ReflectionProperty $property */
            $docComment = $property->getDocComment();
            $docComment = preg_replace('#[ \t]*(?:\/\*\*|\*\/|\*)?[ ]{0,1}(.*)?#', '$1', $docComment);
            $docComment = trim(str_replace('*/', null, $docComment));
            foreach (explode("\n", $docComment) as $item) {
                if (strpos($item, '@inject') !== false) {
                    $inject = trim(str_replace('@inject', null, $item));
                    $value = null;
                    eval("\$value = {$inject};"); // yes, eval. uggly, isnt't?
                    $property->setAccessible(true);
                    $property->setValue($this, $value);
                }
            }
        }
    }

```

 If you have read "Clean Code" (if not, you must do it) you noticed that uncle Bob doesn't like this class. The method is too long, so we are going to refactor a little bit.

```php
class DocInject
{
    public function __construct()
    {
        $reflection = new ReflectionClass($this);
        foreach ($reflection->getProperties() as $property) {
            $this->processProperty($property);
        }
    }
 
    private function processProperty(ReflectionProperty $property)
    {
        $docComment = $this->cleanPhpDoc($property->getDocComment());
        foreach (explode("\n", $docComment) as $item) {
            if ($this->existsInjectDecorator($item)) {
                $this->performDependencyInjection($property, $item);
            }
        }
    }
 
    private function cleanPhpDoc($docComment)
    {
        $docComment = preg_replace('#[ \t]*(?:\/\*\*|\*\/|\*)?[ ]{0,1}(.*)?#', '$1', $docComment);
        $docComment = trim(str_replace('*/', null, $docComment));
        return $docComment;
    }
 
    private function existsInjectDecorator($item)
    {
        return strpos($item, '@inject') !== false;
    }
 
    private function performDependencyInjection(ReflectionProperty $property, $item)
    {
        $injectString = $this->removeDecoratorFromPhpDoc($item);
        $value = $this->compileInjectString($injectString);
        $this->injectValueIntoProperty($property, $value);
    }
 
    private function removeDecoratorFromPhpDoc($item)
    {
        return trim(str_replace('@inject', null, $item));
    }
 
    private function compileInjectString($injectString)
    {
        $value = null;
        eval("\$value = {$injectString};"); // yes, eval. uggly, isnt't?
        return $value;
    }
 
    private function injectValueIntoProperty(ReflectionProperty $property, $value)
    {
        $property->setAccessible(true);
        $property->setValue($this, $value);
    }
}
```

So now we don’t need to pass the new instance of PDO connection in the constructor with DI:

```php
$user = new User();
print_r($user->getInfo(4));
```

It works but there's something that I don't like. We need to extend our User class with DocInject. I like plain classes. Because of that we are going to use the new feature of PHP5.4: [traits](http://php.net/manual/en/language.oop5.traits.php)

Instead of extend our class with DocInject we are going to create: 

```php
trait DocInject
{
    public function parseDocInject()
    {
        $reflection = new ReflectionClass($this);
        foreach ($reflection->getProperties() as $property) {
            $this->processProperty($property);
        }
    }
 
    private function processProperty(ReflectionProperty $property)
    {
        $docComment = $this->cleanPhpDoc($property->getDocComment());
        foreach (explode("\n", $docComment) as $item) {
            if ($this->existsInjectDecorator($item)) {
                $this->performDependencyInjection($property, $item);
            }
        }
    }
 
    private function cleanPhpDoc($docComment)
    {
        $docComment = preg_replace('#[ \t]*(?:\/\*\*|\*\/|\*)?[ ]{0,1}(.*)?#', '$1', $docComment);
        $docComment = trim(str_replace('*/', null, $docComment));
        return $docComment;
    }
 
    private function existsInjectDecorator($item)
    {
        return strpos($item, '@inject') !== false;
    }
 
    private function performDependencyInjection(ReflectionProperty $property, $item)
    {
        $injectString = $this->removeDecoratorFromPhpDoc($item);
        $value = $this->compileInjectString($injectString);
        $this->injectValueIntoProperty($property, $value);
    }
 
    private function removeDecoratorFromPhpDoc($item)
    {
        return trim(str_replace('@inject', null, $item));
    }
 
    private function compileInjectString($injectString)
    {
        $value = null;
        eval("\$value = {$injectString};"); // yes, eval. uggly, isnt't?
        return $value;
    }
 
    private function injectValueIntoProperty(ReflectionProperty $property, $value)
    {
        $property->setAccessible(true);
        $property->setValue($this, $value);
    }
}
```

And now:

```php
class User
{
    use DocInject;
 
    public function __construct()
    {
        $this->parseDocInject();
    }
 
    /**
     * @inject Db::getDb(DbConf::DB1)
     * @var PDO
     */
    private $db;
 
    public function getInfo($uid)
    {
        $sql = "select * from users where uid=:UID";
        $stmp = $this->db->prepare($sql);
        $stmp->execute(array('UID' => $uid));
        return $stmp->fetchAll();
    }
}
```

This implementation has a little problem. If our class User needs a constructor we have a problem. As far as I know we cannot use parent::\_\_construct() with a trait. We can solve this problem changing the code a little bit:

```php
class User
{
    use DocInject {parseDocInject as __construct;}
 
    /**
     * @inject Db::getDb(DbConf::DB1)
     * @var PDO
     */
    private $db;
 
    public function getInfo($uid)
    {
        $sql = "select * from users where uid=:UID";
        $stmp = $this->db->prepare($sql);
        $stmp->execute(array('UID' => $uid));
        return $stmp->fetchAll();
    }
}
```

A simple unit test 

```php
public function testSimple()
{
    $user = new User();
    $this->assertTrue(count($user->getInfo(4)) > 0);
}
```

If we use different DbConf file for each environment we can easily use one database or another without changing any line of code.

And that's all. What do you think?

(Files available as gist [here](https://gist.github.com/2343376) and [here](https://gist.github.com/2343390))
