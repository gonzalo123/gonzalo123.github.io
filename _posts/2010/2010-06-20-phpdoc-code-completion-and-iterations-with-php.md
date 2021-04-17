---
layout: post
title: "PHPDoc, code completion and iterations with PHP"
date: "2010-06-20"
tags: 
  - "php"
categories: 
  - "php"
---

Coding is very important part within our work as developers. Maybe it isn’t the outcome of our work. Our clients don’t really mind if our code is smart or not. They only want the product or service we give  them. Code is our problem. However smart code and agile programming skills are really useful to accomplish with our objectives. One useful tool is **[PHPDoc](http://www.phpdoc.org/)**.

PHP is not Java. Java is a [strongly typed](http://en.wikipedia.org/wiki/Strongly_typed_programming_language) programming language. Because of that modern IDE helps us a lot when we are coding Java. In PHP we don’t need to declare the type of a variable when we declare it. Even we don’t need t declare the variables. This behaviour can be good or bad. For IDEs it's very bad. I’m not going to start the _first_ flame-war between Java and PHP now. The fact is that we want to code as fast as we can making less mistakes and writing the less code as we can to reach our goals. PHP has some limitations but we also have some tools as PHPDoc to help us.

Let’s start with an example

```php
class MyClass
{
    function hello($name)
    {
        return "Hello " . $name . "\n";
    }
}
```

When we want to use this class and we type:

```php
$obj = new MyClass;
echo $obj->       <------ code completion appear in our IDE suggesting hello function

```

But now we put a simple singleton function in MyClass

```php
class MyClass
{
    private static $_instance = null;
    function singleton()
    {
        if (is_null(self::$_instance)) {
            self::$_instance = new self;
        }
        return self::$_instance;
    }
 
    function hello($name)
    {
        return "Hello " . $name . "\n";
    }
}
```

And we use our singleton:

```php
echo MyClass::singleton()->     <---- ups nothing happens
```

That’s because our IDE doesn’t know the return type of our singleton function. Because of that we are going to help to our IDE with a bit of PHPDoc magic.

```php
private static $_instance = null;
/**
 * @return MyClass
 */
function singleton()
{
    if (is_null(self::$_instance)) {
        self::$_instance = new self;
    }
    return self::$_instance;
}
```

And now our IDE will help us.

And now something a bit different. We have other class:

```php
class MyClass2
{
    static function getAll()
    {
        return array(new MyClass, new MyClass, new MyClass);
    }
}
```

According to PHPDoc manual we can hint getAll function with _@return array\[int\]MyClass_ but code completion doesn’t work, at least on my tests (please tell me I’m wrong because it’s the best solution). So I need to use a small trick to get the autocompletion:

```php
foreach (MyClass2::getAll() as $row) {
    /* @var $row MyClass */
    echo $row->     <------------------- code completion appear
}
```

I’ve tried with @return array\[int\]MyClass and also using SPL Iterators but that’s the only way I know to ensure autocompletion.

Other possible solution is use a dummy cast function in MyClass. It’s a bit ugly solution but also works always:

```php
/**
 * @param MyClass $cls
 * @return MyClass
 */
static function cast($cls)
{
    return $cls;
}
```

Now We can use:

```php
foreach (MyClass2::getAll() as $row) {
    echo MyClass::cast($row)->hello("Gonzalo"); // code completion works
}
```
