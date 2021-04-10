---
layout: post
title: "Building a small microframework with PHP"
date: "2011-08-22"
tags: 
  - "php"
  - "technology"
  - "framework"
  - "microframework"
---

Nowadays microframewors are very popular. Since Blake Mizerany created [Sinatra](http://www.sinatrarb.com/) (Ruby), we have a lot of Sinatra clones in PHP world. Probably the most famous (and a really good one) is [Silex](http://silex-project.org/). But we also have several ones, such as [Limonade](http://www.limonade-php.net/), [GluePHP](http://gluephp.com/) and [Slim](http://www.slimframework.com/). Those frameworks are similar. We define routes and we connect those routes to callbacks:

For example:

> '/hello/{name}' => function ($name) {return "Hello $name";}

Those microframeworks use the new PHP 5.3 callback features. It's easy to build prototypes with those frameworks. I've used silex for a small prototype and I'm really happy with it. But I have a little problem. Each time I need to create a new route I need to create the route and create the callback. The business logic is inside the callback, but I need to tell to the framework where is it with the router. That's means code de callback and write the router. This way of work has advantages. You can change the routes without changing the business logic. I feel comfortable with it in small projects, but when it scales it's difficult to manage (at least for me). [Symfony2](http://symfony.com/) has a great way to create [routes](http://symfony.com/doc/current/book/routing.html), with inheritance, catching and things like that, but sometimes I dont't feel confortable with it. Because of that I have done this small microframework experiment. The idea is drop the router and create it depending on the filesystem. The first idea was inside this postit:

[](/assets/images/microframework1.jpg "microFramework")]

Yes I now. If I want to change the class inside the filesystem, I need to change the urls. As an experiment, I've created a small microframework. The idea the following one:

- .htaccess to redirect every request to index.php
- a set of classes (plain php classes)
- index.php will invoke the required class' function
- after invoking the required function index.php will convert the output to the format selected

[](/assets/images/folder.png "folder")]

Basically index.php will follow the following script: 

```php
setUpAutoload();
list($className, $functionName, $format, $params) = decodeUri(getUri());
$realParams = getRealParams($className, $functionName, $params);
echo format($format, call_user_func_array(array($className, $functionName), $realParams));
```

Here you have the full index.php code: 

```php
function call_user_func_named($className, $obj, $function, $params)
{
    $class = new ReflectionClass($obj);
    $reflect = $class->getMethod($function);
 
    $realParams = array();
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
            throw new Exception("{$className}::{$function}() param: missing param: {$pname}");
        }
    }
 
    return call_user_func_array(array(new $obj, $function), $realParams);
}
 
function decodeUri($uri)
{
    $conf = $params = array();
    $functionName = $format = null;
 
    $parsedUrl = parse_url($uri);
    $path = $parsedUrl['path'];
    if (isset($parsedUrl['query'])) {
        $query = $parsedUrl['query'];
        $params = array();
        $pairs = explode('&', $query);
        foreach ($pairs as $pair) {
            if (trim($pair) == '') {
                continue;
            }
            list($key, $value) = explode('=', $pair);
            $params[$key] = urldecode($value);
        }
    }
    $arr = explode('/', $path);
 
    for ($i = 0; $i < count($arr); $i++) {
        $elem = $arr[$i];
        if (strpos($elem, '.') !== false) {
            list($functionName, $format) = explode(".", $elem);
            continue;
        } else {
            if ($elem != '') $conf[] = ucfirst($elem);
        }
    }
 
    $className = implode('\\', $conf);
    return array($className, $functionName, $format, $params);
}
 
function format($format, $out)
{
    switch ($format) {
        case 'json':
            header('Cache-Control: no-cache, must-revalidate');
            header('Expires: Mon, 26 Jul 1997 05:00:00 GMT');
            header('Content-type: application/json');
            return json_encode($out);
        case 'html':
        case 'htm':
            header('Cache-Control: no-cache, must-revalidate');
            header('Expires: Mon, 26 Jul 1997 05:00:00 GMT');
            header('Content-type: Content-Type: text/html');
            return (string)$out;
        case 'txt':
        case 'ajax':
            header('Content-type: text/plain; charset=utf-8');
            return (string)$out;
        case 'css':
            header('Content-type: text/css');
            return (string)$out;
        case 'js':
            header('Content-type: application/javascript');
            return (string)$out;
        case 'jsonp':
            $cbk = filter_input(INPUT_GET, '_cbk', FILTER_SANITIZE_STRING);
            if ($cbk == '') {
                $cbk = 'cbk';
            }
 
            header('Content-type: text/javascript; charset=utf-8');
            return "{$cbk}(" . json_encode($out) . ");";
        default:
            throw new Exception("Undefined format");
    }
}
 
function getRealParams($className, $functionName, $params)
{
    $realParams = array();
    $class = new \ReflectionClass(new $className);
    $reflect = $class->getMethod($functionName);
 
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
            throw new Exception("{$className}::{$functionName}() param: missing param: {$pname}");
        }
    }
    return $realParams;
}
 
function setUpAutoload()
{
    spl_autoload_register(function ($class)
        {
            $class = str_replace('\\', '/', $class) . '.php';
            if (is_file($class)) {
                require_once($class);
            } else {
                throw new Exception("{$class} does not exists");
            }
        }
    );
}
 
function getUri()
{
    $requestUri = $_SERVER['REQUEST_URI'];
    $scriptName = $_SERVER['SCRIPT_NAME'];
 
    if (dirname($scriptName) == '/') {
        $uri = $requestUri;
        return $uri;
    } else {
        $uri = str_replace(dirname($scriptName), null, $requestUri);
        return $uri;
    }
}
```

An example of the plain php classes with the business logic: 

```php
namespace Demo;
 
class Foo
{
    public function hello()
    {
        return "Hello";
    }
 
    public function helloName($name)
    {
        return "Hello " . $name;
    }
 
    public function getUsers()
    {
        return array('Gonzalo', 'Peter', );
    }
}
```

This class is easily testable with PHPUnit.

Normally I use this kind of framework to get json and jsonp. It's pretty straightforward to get json:

> http://localhost/demo/foo/getUsers.json \["Gonzalo","Peter"\] http://localhost/demo/foo/getUsers.jsonp cbk(\["Gonzalo","Peter"\]);

This is a very small approach of what I have in mind, coded with a few lines to show the concept. I'm working in some more advanced features such as twig integration, security, and things like that. The idea is combine the plain PHP classes with annotations (with Addendum). Imagine a class like that:

Here @Render() annotation tells to the framework to use a twig template. 

```php
// http://localhost/tdl/index.html
 
class Tdl
{
    /**
     * @Render()
     * @return array
     */
    public function index()
    {
        return array(
            'title'       => 'Simple ToDo List',
            'description' => 'Simple ToDo List',
            'author'      => 'Gonzalo',
        );
    }
}
```

And here will output a javascript file using assetic library 

```php
// http://localhost/tdl/js/js.js
 
namespace Tdl;
 
use Assetic\Asset\AssetCollection;
use Assetic\Asset\FileAsset;
use Assetic\Filter\FilterInterface;
 
class Js
{
    public function js()
    {
        $js = new AssetCollection(array(
            new FileAsset(MVC_PATH . '/V/Tdl/Js/nf.js'),
            new FileAsset(MVC_PATH . '/V/Tdl/Js/main.js'),
        ));
 
        return  $js->dump();
    }
}
```

Sometimes I think Symfony2 has all this features and developing this framework is a waste of time. Probably, but it's cool to code it. Also it's very easy and fast for me build prototypes. What do you think?

Initial commit [here](https://github.com/gonzalo123/microFramework/tree/post1) (the code used in this post). Full code on [github](https://github.com/gonzalo123/microFramework)
