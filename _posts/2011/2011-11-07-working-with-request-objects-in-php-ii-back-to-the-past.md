---
layout: post
title: "Working with Request objects in PHP (II). Back to the past"
date: "2011-11-07"
tags: 
  - "php"
  - "technology"
---

In one of my last post “[Working with request objects in PHP](http://gonzalo123.wordpress.com/2011/10/17/working-with-request-objects-in-php/ "Working with Request objects in PHP")”, I wrote a simple library to handle request objects. According that post let's do a bit of history of PHP. In the early years of PHP (PHP3 - PHP4) one of the cool features of PHP was the "variable injection" inside our projects with [register\_globals](http://es2.php.net/manual/en/security.globals.php)\=on. If you had the following a url:

> index.php?parameter1=Hi

Your script had magically a variable called $parameter1 with the value “Hi”. This feature has horrible security problems, if our user can inject variables in our scripts, especially with a loose typing program language like PHP. Because of that we all swap from those injected variables to get the value from $\_POST and $\_GET superglobals. In fact "injected variables" are disabled long time ago within PHP configuration.

Nowadays we [don’t](http://www.phparch.com/2010/07/never-use-_get-again/) use $\_POST $\_GET superglobals directly. We need to filter the input. Because of that I wrote [RequestObject](https://github.com/gonzalo123/RequestObject/blob/master/RequestObject.php) library. Now we’re going to back to the past and allow the use of injected variables, but filtered.

RequestObject has now an extra public function called getFilteredParameters. This function returns an array with all already filtered input parameters. So if we use “[extract](http://php.net/manual/function.extract.php)” function we can create variables for each input parameters, but with the filtered values:

```php
class Request extends RequestObject
{
    /** @cast string */
    public $param1;
    /**
     * @cast string
     * @default default value
     */
    public $param2;
}
 
$request = new Request();
extract($request->getFilteredParameters());
 
echo "param1: <br/>";
var_dump($param1);
echo "<br/>";
 
echo "param2: <br/>";
var_dump($param2);
echo "<br/>";
```

Full library available on github [here](https://github.com/gonzalo123/RequestObject/)
