---
layout: post
title: "Working with Request objects in PHP"
date: "2011-10-17"
tags: 
  - "php"
  - "technology"
---

Normally when we work with web applications we need to handle Request objects. Requests are the input of our applications. According to the golden rule of security:

> Filter Input-Escape Output

We cannot use $\_GET and $\_POST superglobals. OK we can use then but we [shouldn’t](http://www.phparch.com/2010/07/never-use-_get-again/) use them. Normally web frameworks do this work for us, but not all is a framework.

Recently I have worked in a small project without any framework. In this case I also need to handle Request objects. Because of that I have built this small library. Let me show it.

Basically the idea is the following one. I want to filter my inputs, and I don't want to remember the whole name of every input variables. I want to define the Request object once and use it everywhere. Imagine a small application with a simple input called param1. The URL will be:

> test1.php?param1=11212

and we want to build this simple script: 

```php
echo "param1: " . $_GET['param1'] . '<p/>';
```

The problem with this script is that we aren't filtering input. And we also need to remember the parameter name is param1. If we need to use param1 parameter in another place we need to remember its name is param1 and not Param1 or para1. It can be obvious but it's easy to make mistakes.

My proposal is the following one. I create a simple PHP class called Request1 extending RequestObject object:

## Example 1: simple example

```php
class Request1 extends RequestObject
{
    public $param1;
}
```

Now if we create an instance of Request1, we can use the following code: 

```php
$request = new Request1();
echo "param1: " . $request->param1 . '<p/>';
```

I'm not going to explain the magic now, but with this simple script we will filter the input to the default type (string) and we will get the following outcomes:

```
test1.php?param1=11212
param1: 11212

test1.php?param1=hi
param1: hi
```

Maybe is hard to explain with words but with examples it's more easy to show you what I want:

## Example 2: data types and default values

```php
class Request2 extends RequestObject
{
    /**
     * @cast string
     */
    public $param1;
    /**
     * @cast string
     * @default default value
     */
    public $param2;
}
 
$request = new Request2();
 
echo "param1: <br/>";
var_dump($request->param1);
echo "<br/>";
 
echo "param2: <br/>";
var_dump($request->param2);
echo "<br/>";
```

Now we are will filter param1 parameter to string and param2 to string to but we will assign a default variable to the parameter if we don't have a user input.

```
test2.php?param1=hi&param2=1

param1: string(2) "hi"
param2: string(1) "1"

test2.php?param1=1&param2=hi

param1: string(1) "1"
param2: string(2) "hi"

test2.php?param1=1
param1: string(1) "1"
param2: string(13) "default value"
```

## Example 3: validadors

```php
class Request3 extends RequestObject
{
    /** @cast string */
    public $param1;
    /** @cast integer */
    public $param2;
 
    protected function validate_param1(&$value)
    {
        $value = strrev($value);
    }
     
    protected function validate_param2($value)
    {
        if ($value == 1) {
            return false;
        }
    }
}
try {
    $request = new Request3();
 
    echo "param1: <br/>";
    var_dump($request->param1);
    echo "<br/>";
 
    echo "param2: <br/>";
    var_dump($request->param2);
    echo "<br/>";
} catch (RequestObjectException $e) {
    echo $e->getMessage();
    echo "<br/>";
    var_dump($e->getValidationErrors());
}
```

Now a complex example. param1 is a string and param2 is an integer, but we also will validate them. We will alter the param1 value (a simple [strrev](http://php.net/manual/en/function.strrev.php)) and we also will raise an exception if param2 is equal to 1

```
test3.php?param2=2&param1=hi
param1: string(2) "ih"
param2: int(2)

test3.php?param1=hola&param2=1
Validation error
array(1) { ["param2"]=> array(1) { ["value"]=> int(1) } }
```

## Example 4: Dynamic validations

```php
class Request4 extends RequestObject
{
    /** @cast string */
    public $param1;
    /** @cast integer */
    public $param2;
}
 
$request = new Request4(false); // disables perform validation on contructor
                               // it means it will not raise any validation exception
$request->appendValidateTo('param2', function($value) {
        if ($value == 1) {
            return false;
        }
    });
 
try {
    $request->validateAll(); // now we perform the validation
 
    echo "param1: <br/>";
    var_dump($request->param1);
    echo "<br/>";
 
    echo "param2: <br/>";
    var_dump($request->param2);
    echo "<br/>";
} catch (RequestObjectException $e) {
    echo $e->getMessage();
    echo "<br/>";
    var_dump($e->getValidationErrors());
}
```

More complex example. Param1 will be cast as string and param2 as integer again, same validation to param2 (exception if value equals to 1), but now validation rule won't be set in the definition of the class. We will append dynamically after the instantiation of the class.

```
test4.php?param1=hi&param2=2
param1: string(4) "hi"
param2: int(2)

test4.php?param1=hola&param2=1
Validation error
array(1) { ["param2"]=> array(1) { ["value"]=> int(1) } }
```

## Example 5: Arrays and default params

```php
class Request5 extends RequestObject
{
    /** @cast arrayString */
    public $param1;
 
    /** @cast integer */
    public $param2;
 
    /**
     * @cast arrayString
     * @defaultArray "hello", "world"
     */
    public $param3;
 
    protected function validate_param2(&$value)
    {
        $value++;
    }
}
 
$request = new Request5();
 
echo "<p>param1: </p>";
var_dump($request->param1);
 
echo "<p>param2: </p>";
var_dump($request->param2);
 
echo "<p>param3: </p>";
var_dump($request->param3);
```

Now a simple example but input parameters allow arrays and default values.

```
test5.php?param1[]=1&param1[]=2&param2[]=hi
param1: array(2) { [0]=> int(1) [1]=> int(2) }
param2: int(1)
param3: array(2) { [0]=> string(5) "hello" [1]=> string(5) "world" }

test5.php?param1[]=1&param1[]=2&param2=2
param1: array(2) { [0]=> string(1) "1" [1]=> string(1) "2" }
param2: int(3)
param3: array(2) { [0]=> string(5) "hello" [1]=> string(5) "world" }
```

## RequestObject

The idea of RequestObject class is very simple. When we create an instance of the class (in the constructor) we filter the input request (GET or POST depending on REQUEST\_METHOD) with [filter\_var\_array](http://php.net/manual/en/function.filter-var-array.php) and [filter\_var](http://es2.php.net/manual/en/function.filter-var.php) functions according to the rules defined as annotations in the RequestObject class. Then we populate the member variables of the class with the filtered input. Now we can use to the member variables, and auto-completion will work perfectly with our favourite IDE with the parameter name. OK. I now. I violate encapsulation principle allowing to access directly to the public member variables. But IMHO the final result is more clear than creating an accessor here. But if it creeps someone out, we would discuss another solution :).

Full code here on [github](https://github.com/gonzalo123/RequestObject)

What do you think?
