---
layout: post
title: "Building a simple Dependency Injection Container with PHP"
date: "2012-07-09"
tags: 
  - "dependency-injection"
  - "php"
  - "technology"
  - "di"
---

If you are looking for a small Dependency Injection Container with PHP maybe you need have look to **Pimple**.

> [Pimple](http://pimple.sensiolabs.org/) is a small Dependency Injection Container for PHP 5.3 that consists of just one file and one class (about 50 lines of code).

Now, keeping with my aim of reinvent the wheel, we will create a simple Dependency Injection Container basically to understand how does it work. Let’s start.

First of all: Do we really need a Dependency Injection Container (DIC)? If you are asking yourself this question, maybe you need to have look to Fabien Poetencier’s [article](http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container).

We are going to work with a teorical problem like this:

Imagine we are going to build a service that uses one external REST API. We define your application with three classes:

- **App**. The main application.
- **Proxy** The part of the application that speaks with the external API
- **Curl**. One curl wrapper to perform our http connections to the REST API

Our first approach can be:

```php
<?php
class App
{
    private $proxy;
 
    public function __construct()
    {
        echo "App::__construct\n";
        $this->proxy = new Proxy();
    }
 
    public function hello()
    {
        return $this->proxy->hello();
    }
}
 
class Proxy
{
    private $curl;
 
    public function __construct()
    {
        $this->curl = new Curl();
    }
 
    public function hello()
    {
        echo "Proxy::__construct\n";
        return $this->curl->doGet();
    }
}
 
class Curl
{
    public function doGet()
    {
        echo "Curl::doGet\n";
        return "Hello";
    }
}
 
$app = new App();
echo $app->hello();
```

If we execute the script:

```shell
php example1.php 
 
App::__construct
Proxy::__construct
Curl::doGet
Hello
```

It works but we have one problem. Our application is strongly coupled. As we can see App creates a new instance of Proxy within the constructor and Proxy creates a new instance of Curl. That’s a problem especially if we want to use TDD. What happens if we want to mock Curl requests to test the application without using the real external service?. **Dependency injection** can help us here. We can change our application to:

```php
<?php
class App
{
    private $proxy;
 
    public function __construct(Proxy $proxy)
    {
        echo "App::__construct\n";
        $this->proxy = $proxy;
    }
 
    public function hello()
    {
        return $this->proxy->hello();
    }
}
 
class Proxy
{
    private $curl;
 
    public function __construct(Curl $curl)
    {
        $this->curl = $curl;
    }
 
    public function hello()
    {
        echo "Proxy::__construct\n";
        return $this->curl->doGet();
    }
}
 
class Curl
{
    public function doGet()
    {
        echo "Curl::doGet\n";
        return "Hello";
    }
}
 
 
$app = new App(new Proxy(new Curl()));
echo $app->hello();
```

The outcome is exactly the same but now we can easily use mocks and use different configurations depending on the environment. Maybe your testing development does not have access to the real REST server.

Now our application isn’t coupled but as we can see our Dependency Injection becomes a mess. That’s one problem with DI. It’s pretty straightforward to inject simple things but when we have dependencies over a set of classes that’s becomes a difficult task. Because of that we can use **Dependency Injection Containers**.

If we choose **Pimple** as Dependency Injection Container we can refactor our application to:

```php
<?php
class App
{
    private $proxy;
 
    public function __construct(Pimple $container)
    {
        echo "App::__construct\n";
        $this->proxy = $container['Proxy'];
    }
 
    public function hello()
    {
        return $this->proxy->hello();
    }
}
 
class Proxy
{
    private $curl;
 
    public function __construct(Pimple $container)
    {
        $this->curl = $container['Curl'];
    }
 
    public function hello()
    {
        echo "Proxy::__construct\n";
        return $this->curl->doGet();
    }
}
 
class Curl
{
    public function doGet()
    {
        echo "Curl::doGet\n";
        return "Hello";
    }
}
 
require_once 'Pimple.php';
 
$container = new Pimple();
$container['Curl'] = function ($c) {return new Curl();};
$container['Proxy'] = function ($c) {return new Proxy($c);};
 
$app = new App($container);
echo $app->hello();
```

But what is my problem with Pimple? Basically my problem is that my IDE cannot autocomplete correctly $container is an instance of Pimple not the “real” instance. OK It instantiated on demand the classes but it’s done at runtime and the IDE don’t know about that. We can solve it using an extra PHPDoc to give hints to the IDE but we also can use a different approach. Instead of using Pimple we can use this script:

```php
<?php
class App
{
    private $proxy;
 
    public function __construct(Container $container)
    {
        echo "App::__construct\n";
        $this->proxy = $container->getProxy();
    }
 
    public function hello()
    {
        echo "App::hello\n";
        return $this->proxy->hello();
    }
}
 
class Proxy
{
    private $curl;
 
    public function __construct(Container $container)
    {
        echo "Proxy::__construct\n";
        $this->curl = $container->getCurl();
    }
 
    public function hello()
    {
        return $this->curl->doGet();
    }
}
 
class Curl
{
    public function doGet()
    {
        echo "Curl::doGet\n";
        return "Hello";
    }
}
 
class Container
{
    public function getProxy()
    {
        return new Proxy($this);
    }
 
    public function getCurl()
    {
        return new Curl();
    }
}
 
$app = new App(new Container());
echo $app->hello();
```

The idea is the same than Pimple but now we have created our custom Dependency Injection Container without extending any library and now our IDE will autocomplete the fucntion names without problems. If we want to share objects instead creating new ones each time we call the factory function of the container we can change a little bit our Container (the same way than Pimple::share) with a simple singleton pattern:

```php
class Container
{
    static $proxy;
    public function shareProxy()
    {
        if (NULL === self::$proxy) self::$proxy = new Proxy($this);
        return self::$proxy;
    }
 
    public function getCurl()
    {
        return new Curl();
    }
}
```

And that’s all. What do you think?
