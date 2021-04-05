---
layout: post
title: "Deployment tip: How to use environ variables to create different environments with PHP"
date: "2012-08-27"
tags: 
  - "php"
  - "technology"
  - "apache"
---

If you use a framework such as Symfony2 this problem is solved for you, but if you aren't using any framework you probably need to solve it in one way or another. Let me explain it. One typical scenario: production and development. We have one development database and another one for production. Each database has it's own connection string. Probably we need to hard-code the connection string within the PHP code, but obviously if we are in the development environment we are going to use one connection string and the production one in the production environment. We can solve the problem with exotic tricks in the deployment script. I like to use exactly the same source code in all environments. Exactly the same means exactly the same, so I cannot change the code before pushing it to production. Mainly because normally my production environment usually it's a cloud, and change the code it's a mess. What can we do?

The solution that I like for this kind of problems is to use apache's environ variables. We inject the environ variables in the virtual host configuration:

```xml
<VirtualHost *:80>
    ...
    SetEnv GONZALO_ENVIRON development
    ...
</VirtualHost>
```

Now we can read the environ variable easily with PHP with:

```php
$environ = getenv('GONZALO_ENVIRON');
```

I've seen people who use this trick to avoid to hard-code our database passwords and connection string in the PHP souce code. Using this the developers cannot see the passwords (only sysadmins). I don't like it, basically because I'm a hybrid between developer and sysadmin and this method is not agile for me, but maybe it can be useful for you.

I will show you a little example using this technique. 

```php
<?php
include(__DIR__ . '/../lib/App.php');
 
$environ = getenv('GONZALO_ENVIRON');
$app = new App($environ);
echo $app->run();
```

We have two different configuration files one for development: 

```php
<?php
return array(
    'ENVIRON' => 'DEVELOPMENT',
    'DB'      => array(
        'MAIN' => array(
            'dsn'      => 'pgsql:host=127.0.0.1;port=5432;dbname=dev_dbname',
            'username' => 'devel_username',
            'password' => 'devel_password',
            'options'  => array(
                PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_NAMED
            )
        )
    )
);
```

And another one for production \

```php
return array(
    'ENVIRON' => 'PRODUCTION',
    'DB'      => array(
        'MAIN' => array(
            'dsn'      => 'pgsql:host=xxx.xxx.xxx.xxx;port=5432;dbname=prod_dbname',
            'username' => 'prod_username',
            'password' => 'prod_password',
            'options'  => array(
                PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_NAMED
            )
        )
    )
);
```

Our app library: 

```php
class AppException extends Exception{}
 
class App
{
    private $conf;
 
    public function __construct($environ)
    {
        $this->conf = $this->getConfFromEnviron($environ);
    }
 
    private function getConfFromEnviron($environ)
    {
        $filePath = $this->getConfFilePath($environ);
        if (!is_file($filePath)) {
            throw new AppException("File '{$filePath}' not found");
        }
        return require($filePath);
    }
 
    private function getConfFilePath($environ)
    {
        return __DIR__ . "/../conf/{$environ}.php";
    }
 
    public function getFromConf($key)
    {
        $out = $this->conf;
        foreach (explode('.', $key) as $item) {
            if (isset($out[$item])) {
                $out = $out[$item];
            } else {
                throw new AppException("Key '{$key}' not found");
            }
        }
        return $out;
    }
 
    public function run()
    {
        $environ = $this->getFromConf('ENVIRON');
        return "Hello from {$environ}";
    }
}
```

But probably the best way to explain it is with the tests: 

```php
class AppTest extends PHPUnit_Framework_TestCase
{
    public function testEnvironDevelopment()
    {
        $environ = 'development';
        $app     = new App($environ);
        $this->assertEquals('Hello from DEVELOPMENT', $app->run());
    }
 
    public function testEnvironProduction()
    {
        $environ = 'production';
        $app     = new App($environ);
        $this->assertEquals('Hello from PRODUCTION', $app->run());
    }
 
    /**
     * @expectedException AppException
     */
    public function testEnvironNotFound()
    {
        $environ = 'xxxxx';
        $app     = new App($environ);
    }
 
    public function testGetConf()
    {
        $environ = 'development';
        $app     = new App($environ);
 
        $this->assertEquals('DEVELOPMENT', $app->getFromConf('ENVIRON'));
        $this->assertEquals('devel_username', $app->getFromConf('DB.MAIN.username'));
        $this->assertEquals('devel_password', $app->getFromConf('DB.MAIN.password'));
    }
 
    /**
     * @expectedException AppException
     */
    public function testGetConfWithKeyNotFound()
    {
        $environ = 'development';
        $app     = new App($environ);
 
        $app->getFromConf('DB.MAIN.xxx');
    }
}
```

You can see the whole code at [github](https://github.com/gonzalo123/deploymentTips)
