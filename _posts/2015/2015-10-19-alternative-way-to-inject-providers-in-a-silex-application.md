---
layout: post
title: "Alternative way to inject providers in a Silex application"
date: "2015-10-19"
tags: 
  - "dic"
  - "pimple"
  - "service"
  - "silex"
  - "dependency-injection"
  - "php"
  - "technology"
---

I normally use [Silex](http://silex.sensiolabs.org/) when I need to build one Backend. It's simple and straightforward to build one API endpoint using this micro framework. But there's something that I don't like it: The "array access" way to access to the dependency injection container. I need to remember what kind of object provides my service provider and also my IDE doesn't help me with autocompletion. OK I can use PHPDoc comments or even create one class that inherits from Silex\\Application and use Traits. Normally I'm lazy to do it. Because of that I've create this simple service provider to help me to do what I'm looking for. Let me explain it a little bit.

Imagine that I've got this class 

```php
namespace Foo
 
class Math
{
    public function sum($i, $j)
    {
        return $i+$j;
    }
}
```

And I want to add this service to my DIC 

```php
$app['math'] = $app->share(function () {
    return new Math();
});
```

Now I can use my service within my Silex application

```php
$app->get("/", function () use ($app) {
    return $app['math']->sum(1, 2);
});
```

But I want to use my service in the same way that I'm using my services within my [AngularJS](https://angularjs.org/) applications. I what to do something like that:

```php
use Foo\Math;
...
$app->get("/", function (Math $math) {
    return $math->sum(1, 2);
});
```

And that's exactly what my service provider does. I only need to append my provider to my Application and tell to the provider what's the relationship between Pimple's services keys and its provided Instance

```php
$app->register(new InjectorServiceProvider([
    'Foo\Math' => 'math',
]));
```

This is one example

```commandline
composer require gonzalo123/injector
```

```php
include __DIR__ . "/../vendor/autoload.php";
 
use Silex\Application;
use Injector\InjectorServiceProvider;
use Foo\Math;
 
$app            = new Application(['debug' => true]);
 
$app->register(new InjectorServiceProvider([
    'Foo\Math' => 'math',
]));
 
$app['math'] = function () {
    return new Math();
};
 
$app->get("/", function (Math $math) {
    return $math->sum(1, 2);
});
 
$app->run();
```

And this is the Service Provider

```php
namespace Injector;
use Silex\Application;
use Silex\ServiceProviderInterface;
use Symfony\Component\HttpKernel\Event\FilterControllerEvent;
use Symfony\Component\HttpKernel\KernelEvents;
class InjectorServiceProvider implements ServiceProviderInterface
{
    private $injectables;
    public function __construct($injectables = [])
    {
        $this->injectables = $injectables;
    }
    public function appendInjectables($providedClass, $key)
    {
        $this->injectables[$providedClass] = $key;
    }
    public function register(Application $app)
    {
        $app->on(KernelEvents::CONTROLLER, function (FilterControllerEvent $event) use ($app) {
            $reflectionFunction = new \ReflectionFunction($event->getController());
            $parameters         = $reflectionFunction->getParameters();
            foreach ($parameters as $param) {
                $class = $param->getClass();
                if ($class && array_key_exists($class->name, $this->injectables)) {
                    $event->getRequest()->attributes->set($param->name, $app[$this->injectables[$class->name]]);
                }
            }
        });
    }
    public function boot(Application $app)
    {
    }
}
```

As we can see I'm listening to CONTROLLER event from event dispatcher and I inject the dependency form container to requests attributes.

Full code in my [github](https://github.com/gonzalo123/injector) account
