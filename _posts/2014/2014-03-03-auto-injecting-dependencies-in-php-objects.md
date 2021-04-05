---
layout: post
title: "Auto injecting dependencies in PHP objects"
date: "2014-03-03"
tags: 
  - "dic"
  - "pimple"
  - "dependency-injection"
  - "php"
  - "symfony"
---

I must admit I don't really know what's the correct title for this post. Finally I use "Auto injecting dependencies in PHP objects". I know it isn't very descriptive. Let me explain it a little bit. This time I want to automate the Hollywood Principle ("Don't call us, we'll call you"). The idea is simple. Imagine one "controller"

```php
class Controller
{
    public function hi($name)
    {
        return "Hi $name";
    }
}
```

We can easily automate the "hi" action

```php
$controller = new Controller();
echo $controller->hi("Gonzalo");
```

Or maybe if we are building a framework and our class name and action name depends on user-input: 

```php
$class = "Controller";
$action = "hi";
$arguments = ['name' => "Gonzalo"];
 
echo call_user_function_array([new $class, $action], arguments);
```

But imagine that we want to allow something like that:

```php
class Controller
{
    public function hi($name, Request $request)
    {
        return "Hi $name " .$request->get('surname');
    }
}
```

Now we need to inject Request object within our action "hi", but not always. Only when user set a input variable with the type "Request". Imagine that we also want to allow this kind of injection in the constructor too. We can need to use Reflection to create our instance and to call our action. Sometimes I need to work with custom frameworks and legacy PHP applications. I've done it in a couple of projects, but now I want to create a library to automate this operation.

The idea is to use a Dependency Injection Container ([Pimple](http://pimple.sensiolabs.org/) in my example) and retrieve the dependency from container (if it's available). I cannot use "new" keyword to create the instance and also I cannot call directly the action.

One usage example is: 

```php
class Foo
{
    public function hi($name)
    {
        return "Hi $name";
    }
}
 
class Another
{
    public function bye($name)
    {
        return "Bye $name";
    }
}
 
class Bar
{
    private $foo;
 
    public function __construct(Foo $foo, $surname = null)
    {
        $this->foo     = $foo;
        $this->surname = $surname;
    }
 
    public function hi(Another $another, $name)
    {
        return $this->foo->hi($name . " " . $this->surname) . ' ' . $another->bye($name);
    }
}
 
$container = new Pimple();
$container['name'] = "Gonzalo2";
 
$builder = new G\Builder($container);
 
$bar = $builder->create('Bar', ['surname' => 'Ayuso']);
var_dump($builder->call([$bar, 'hi']));
 
var_dump($bar->hi(new Another(), 'xxxxx'));
```

Our library tries to retrieve the dependecy from the DIC. If it cannot do it, it creates the a new instance. The whole "magic" is in the [Builder](https://github.com/gonzalo123/new/blob/master/src/G/Builder.php) class. You can see the [library](https://github.com/gonzalo123/new) in my github account.
