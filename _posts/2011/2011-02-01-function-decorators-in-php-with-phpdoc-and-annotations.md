---
layout: post
title: "Function decorators in PHP with PHPDoc and Annotations"
date: "2011-02-01"
tags: 
  - "php"
  - "technology"
  - "tips"
---

This days I’m playing with annotations in PHP. PHP, at least just now, doesn’t support annotations natively like Java. I hope it will be available in next versions but if we want to use them already, we need to create “fake” annotations within PHPDoc. We also have another problem. The PHP’s built-in's reflection functions cannot parse properly PHPDoc's annotations but hopefully we have a great library called [addendum](http://code.google.com/p/addendum/) to do this.

Last year I posted an [article](http://gonzalo123.wordpress.com/2009/12/09/things-i-miss-in-php-function-decorators/) called “Things I miss in PHP: Function decorators”. I wanted to solve something that I miss in PHP. Function decorators. A great feature we've got in [python](http://www.python.org/dev/peps/pep-0318/) world. Now I will try to simulate function decorators in PHP with annotations. Let’s start:

The idea is to solve the same problem I had when I wrote the previous article. I want to protect the execution of certain functions in a class to only being executed if the user is logged on. The functions are public. In the previous solution I my classes implement one interface or another if I wanted to ensure that the functions are really public for everyone or if user need to be logged on. The solution with interfaces is clean and simple (no extra libraries need and no reflection need too). The problem is that I can use it only for all the functions of the class. If I want to mark only one function as “only for logged users” I need to divide the class into two classes, one implementing one interface and the second one another interface. With function decorators I can mark only one function within a class. Let’s show the code:

Imagine we have the following class. We are going to play with decorators in the function hello:

```php
class Dummy
{
    public function gonzalo()
    {
        echo "decorator\n";
        return true;
    }
 
    public function gonzalo2()
    {
        echo "decorator2\n";
        return true;
    }
 
    public function gonzalo3()
    {
        echo "post2\n";
        return true;
    }
 
    /** 
     * @PreDispatch("gonzalo") 
     * @PreDispatch("gonzalo2") 
     * @PostDispatch("gonzalo3") 
     * */
    public function hello($name)
    {
        echo "Hello {$name}";
    }
}
```

And if we include the following library (I called NewNew)

```php
include "addendum/annotations.php";
 
class PreDispatch extends Annotation {}
class PostDispatch extends Annotation {} 
 
class NewNew
{
    private $_instance;
 
    function __construct($instance)
    {
        $this->_instance = $instance;
    }
 
    static function factory($instance)
    {
        return new self($instance);
    }
 
    function __call($method, $args)
    {
        $reflectionMethod = new ReflectionAnnotatedMethod(get_class($this->_instance), $method);
        $continue = true;
        $out = null;
        foreach ($reflectionMethod->getAllAnnotations('PreDispatch') as $decorator) {
            $continue = call_user_func_array(array($this->_instance, $decorator->value), $args);
            if ($continue !== true) break;
        }
         
        if ($continue === true) {
            $out = call_user_func_array(array($this->_instance, $method), $args);
         
            foreach ($reflectionMethod->getAllAnnotations('PostDispatch') as $decorator) {
                $continue = call_user_func_array(array($this->_instance, $decorator->value), $args);
                if ($continue === false) break;
            }
        }
 
        return $out;
    }
}
```

As you can see the idea is to call functions annotated as PreDispatch before calling the real function and annotated as PostDispatch before. Those decorator function must return true. If the return value in not true the execution flow will break and next calls will be disabled

And now if we execute the following code:

```php
/** @var Dummy */
$dummy = new NewNew(new Dummy);
echo $dummy->hello("Gonzalo");
```

we will get:

```
decorator
decorator2
Hello Gonzalo
post2
```

OK my decorators are not exactly the same than Phython’s ones, but they meet my requirements.

Now we are going to implement another example.

```php
abstract class Decorators
{
    public function check($user)
    {
        if ($user == 'gonzalo') {
            return true;
        } else {
            return false;
        }
    }
}
class Dummy2 extends Decorators
{
    /** 
     * @PreDispatch("check") 
     * */
    public function hello($name)
    {
        return "Hello {$name}\n";
    }
}
```

And now if we execute the following script:

```php
/** @var Dummy */
$dummy2 = new NewNew(new Dummy2);
echo $dummy2->hello("gonzalo");
echo $dummy2->hello("gonzalo1");
```

we will get

```commandline
$ php decorators.php
Hello gonzalo
```

instead of

```commandline
$ php decorators.php 
Hello gonzalo
Hello gonzalo1
```

Yes. I know. There is a problem with my implementation. The decorators indeed are public functions too and users can call them. They must be protected functions instead of public ones. I need to play a little bit with reflection in my NewNew class and allow to use protected functions instead of public. I’m working on it but i prefer to show the code “as is” now because is more clear to show the idea, What do you think?
