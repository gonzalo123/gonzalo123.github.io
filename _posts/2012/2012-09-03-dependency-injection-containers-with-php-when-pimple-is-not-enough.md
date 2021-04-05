---
layout: post
title: "Dependency Injection Containers with PHP. When Pimple is not enough."
date: "2012-09-03"
tags: 
  - "dependency-injection"
  - "php"
  - "symfony"
  - "technology"
  - "component-library"
  - "di"
  - "language-php"
---

Two months ago I wrote an [article](http://gonzalo123.wordpress.com/2012/07/09/building-a-simple-dependency-injection-container-with-php/) about Dependency Injection with PHP and [Pimple](http://pimple.sensiolabs.org/). After the post I was speaking about it with a group of colleagues and someone threw a question:

> What happens if your container grows up? Does Pimple scale well?

The answer is not so easy. Pimple is really simple and good for small projects, but it becomes a little mess when we need to scale. Normally when I face a problem like that I like to checkout the Symfony2 [code](https://github.com/symfony/symfony). It usually implements a good solution. There's something I really like from Symfony2: it's a brilliant component library and we can use those [components](http://symfony.com/components) within our projects instead of using the full stack framework. So why don't we only use the Dependency Injection [component](https://github.com/symfony/DependencyInjection) from SF2 instead Pimple to solve the problem? Let's start:

We are going to use [composer](http://getcomposer.org/) to load our dependencies so we start writing our composer.json file. We want to use yaml files so we need to add "symfony/yaml" and "symfony/config" in addition to "symfony/dependency-injection":

```json
{
    "require": {
        "symfony/dependency-injection": "dev-master",
        "symfony/yaml": "dev-master",
        "symfony/config": "dev-master"
    },
    "autoload":{
        "psr-0":{
            "":"lib/"
        }
    }
}
```

Now we can run "composer install" command and we already have our vendors and our autolader properly set.

We are going to build exactly the same example than in the previous [post](http://gonzalo123.wordpress.com/2012/07/09/building-a-simple-dependency-injection-container-with-php/):

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
```

Now we create our file "services.yml" describing our dependency injection container behaviour:

```yaml
services:
  app:
    class:     App
    arguments: [@Proxy]
  proxy:
    class:     Proxy
    arguments: [@Curl]
  curl:
    class:     Curl
```

and finally we can build the script:

```php
<?php
include __DIR__ . "/vendor/autoload.php";
 
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
 
$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.yml');
 
$container->get('app')->hello();
```

IMHO is as simple a Pimple but much more flexible, customizable and it's also well documented. For example we can split our yaml files into different ones and load them: 

```php
<?php
include __DIR__ . "/vendor/autoload.php";
 
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
 
$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services1.yml');
$loader->load('services2.yml');
$container->get('app')->hello();
```

An we also can use imports in our yaml files: 

```yaml
imports:
  - { resource: services2.yml }
services:
  app:
    class:     App
    arguments: [@Proxy]
```

If you don't like yaml syntax and you prefer XML (I know. It looks insane :)) you can use it, or even programatically with PHP.

What do you think?

Source code in [github](https://github.com/gonzalo123/dependency-injection) (see the README to install the vendors)
