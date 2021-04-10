---
layout: post
title: "Building a small microframework with PHP (Part 2). Command line interface"
date: "2011-08-29"
tags: 
  - "php"
  - "technology"
  - "cli"
  - "microframework"
---

In my [last post](http://gonzalo123.wordpress.com/2011/08/22/building-a-small-microframework-with-php/) we spoke about building a small microframework with PHP. The main goal of this kind of framework was to be able to map urls to plain PHP classes and become those classes easily testeable with PHPUnit. Now we're going to take a step forward. Sometimes we need to execute the script from command line, for example if we need to use them in a crontab file. OK. We can use curl and perform a simple http call to the webserver. But it's pretty straightforward to create a command line interface (CLI) for our microframework. Zend Framework and Symfony has a great console tools. I've used [Zend\_Console\_Getopt](http://framework.zend.com/manual/en/zend.console.getopt.html) in a project and is really easy to use and well documented. But now we're going to build a command line interface from scratch.

We are going to reuse a lot of code from the earlier post, because of that we are going to encapsulate the code in a class ([DRY](http://goo.gl/QpDP)).

We will use the [getopt](http://php.net/manual/es/function.getopt.php) function from PHP's function set.

```php
$sortOptions = "";
$sortOptions .= "c:";
$sortOptions .= "f:";
$sortOptions .= "v::";
$sortOptions .= "h::";
 
$options = getopt($sortOptions);
```

Then we need to take the paramaters from $GLOBALS['argv'] superglobal according with the options. I use the following hack to prepare $GLOBALS['argv']:

```php
foreach( $options_array as $o => $a ) {
    while($k=array_search("-". $o. $a, $GLOBALS['argv'])) {
        if($k) {
            unset($GLOBALS['argv'][$k]);
        }
    }
    while($k=array_search("-" . $o, $GLOBALS['argv'])) {
        if($k) {
            unset($GLOBALS['argv'][$k]);
            unset($GLOBALS['argv'][$k+1]);
        }
    }
}
$GLOBALS['argv'] = array_merge($GLOBALS['argv']);
```

And now we get the params into $param\_arr array.

```php
$param_arr = array();
$lenght = count((array)$GLOBALS['argv']);
if ($lenght > 0) {
    for ($i = 1; $i < $lenght; $i++) {
        if (isset($GLOBALS['argv'][$i])) {
            list($paramName, $paramValue) = explode("=", $GLOBALS['argv'][$i], 2);
            $param_arr[$paramName] = $paramValue;
        }
    }
}
```

Now we can get className and functionName:

```php
$className    = !array_key_exists('c', $options) ?: $options['c'];
$functionName = !array_key_exists('f', $options) ?: $options['f'];
```

We add a "usage" parameter:

```php
if (array_key_exists('h', $options)) {
     $usage = <<<USAGE
Usage: cli [options] [-c] <class> [-f] <function> [args...]
 
Options:
-h Print this help
-v verbose mode
\n
USAGE;
    echo $usage;
    exit;
}
```

Now we can invoke the function with call\user_func_array

```php
return call_user_func_array(array($className, $functionName), $this->realParams);
```

As you can see, instead of using $param\_arr as parameters array, We need to create an extra $realParams array. The aim of this realParams arrays is to use call\_user\_func\_array with named parameters. In getRealParams function we use reflection to see what are the real parameters of our function and use only those parameters form in the correct order instead. With this trick we will allow to the user to use the parameters in the his desired order, and without forcing to use the real order of our function.

```php
private function getRealParams($params)
{
    $realParams = array();
    $class = new ReflectionClass(new $this->className);
    $reflect = $class->getMethod($this->functionName);
 
    foreach ($reflect->getParameters() as $i => $param) {
        $pname = $param->getName();
        if ($param->isPassedByReference()) {
            /// @todo shall we raise some warning?
        }
        if (array_key_exists($pname, $params)) {
            $realParams[] = $params[$pname];
        } else if ($param->isDefaultValueAvailable()) {
            $realParams[] = $param->getDefaultValue();
        } else {
            throw new Exception("{$this->className}::{$this->functionName}() param: missing param: {$pname}");
        }
    }
    return $realParams;
}
```

And now we can use our microframework from the command line: 

```commandline
./cli -c 'Demo\Foo' -f hello
Hello
 
./cli -c Demo\\Foo -f getUsers
["Gonzalo","Peter"]
 
./cli -c Demo\\Foo -f helloName name=Gonzalo surname=Ayuso
Hello Gonzalo Ayuso
```

Full code on [github](https://github.com/gonzalo123/microFramework)
