---
layout: post
permalink: /:year/:month/:day/:title/
title: "Moving singleton and factory patterns to Abstract with php 5.3"
date: "2009-12-07"
categories: 
  - "php"
  - "web-development"
---

I have built a backend library. I have a tree of classes and i want to use [singleton](http://en.wikipedia.org/wiki/Singleton_pattern "singleton") and [factory](http://en.wikipedia.org/wiki/Factory_method_pattern "factory") patterns to my class set. Easy isn't it?

My dummy class

```php
class Lib_Myclass
{
    public function function1($var1)
    {
        return $var1;
    }
}
```

new instance of my class

```php
$obj = new Lib_Myclass;
$obj->function1('hi');
```

Ok. Nice OO tutorial Gonzalo, but I want to use factory pattern so:

```php
class Lib_Myclass
{
    static function factory()
    {
        return new Lib_Myclass;
    }
    public function function1($var1)
    {
        return $var1;
    }
}
 
Lib_Myclass::factory()->function1('hi');
```

Now imagine you have a lot of classes. You must create over and over the factory function in every classes. OK you can create an abstract class and extend all classes with your abstract class but ...

```php
abstract class AbstractClass
{
    static function factory()
    {
        return new Lib_Myclass; // <---- what's the name of the class?
    }
}
```

Whats is the name of the class you will use when creating the new object? You can use _CLASS but it only works when you are in the parent class. In an extended class CLASS points to the name of the abstract class and not the parent one. With php < 5.3 there are some tricks for doing it but those tricks are very ugly. In php 5.3 we have [get\_called\_class](http://php.net/manual/en/function.get-called-class.php "get_called_class"). So now we can create an abstract class with or factory and singleton implementations and extend our classes without adding any extra code over and over again

```php
abstract class AbstractClass
{
    static $_instance = null;
 
    static function singleton()
    {
        if (is_null(self::$_instance)) {
            self::$_instance = self::factory();
        }
        return self::$_instance;
    }
 
    static function factory()
    {
        $class = get_called_class();
        return new $class;
    }
}
```

And now or class:

```php
class Lib_Myclass extends AbstractClass
{
    public function function1($var1)
    {
        return $var1;
    }
}
```

And use patterns:

```php
<?php
Lib_Myclass::factory()->function1('hi');
 
Lib_Myclass::singleton()->function1('hi');
```

Clean and simple. I like it.
