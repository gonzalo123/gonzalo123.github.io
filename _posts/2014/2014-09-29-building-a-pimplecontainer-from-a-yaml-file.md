---
layout: post
title: "Building a Pimple/Container from a YAML file"
date: "2014-09-29"
tags: 
  - "php"
  - "symfony"
  - "technology"
  - "config"
  - "container"
  - "dic"
  - "pimple"
  - "solid"
  - "yaml"
---

The last May I attended to the great [deSymfony day](http://day.desymfony.com/) conference in Barcelona. At speaker's dinner I had a great conversation with [Máximo Cuadros](https://twitter.com/mcuadros_) about Dependency Injection Containers. We discuss about the customisation of containers. I said that I prefer Symfony´s DIC instead of Pimple, mainly because its configuration with YAML (or even xml) files. But In fact we can customise [Pimple/Containers](http://pimple.sensiolabs.org/) with YAML files in a similar way than we do it with Symfony's DIC. In this example we're going to see one way to do it.

We can easily extend the Pimple/Container and add a function to load a YAML files, parse them and build the container. But doing this we're violating various SOLID principles. First we're violating the Open-Close principle, because to extend our Pimple/Container with the new functionality we are adding new code within an existing class. We're also violating the Dependency Inversion Principle and our new Pimple/Container is going to be harder to maintain. And finally we're obviously violating the Single Responsibility Principle, because our new Pimple/Container is not only a DIC, it's also a YAML parser.

There's another way to perform this operation without upsetting SOLID principles. We can use the Symfony's Config [component](http://blog.servergrove.com/2014/02/21/symfony2-components-overview-config/)

The idea is simple. Imagine this simple application: 

```php
use Pimple\Container;
 
$container = new Container();
$container['name'] = 'Gonzalo';
 
$container['Curl'] = function () {
    return new Curl();
};
$container['Proxy'] = function ($c) {
    return new Proxy($c['Curl']);
};
 
$container['App'] = function ($c) {
    return new App($c['Proxy'], $c['name']);
};
 
$app = $container['App'];
echo $app->hello();
```

We define the dependencies with code. But we want to define dependencies using a yml file for example:

```yaml
parameters:
  name: Gonzalo
 
services:
  App:
    class:     App
    arguments: [@Proxy, %name%]
  Proxy:
    class:     Proxy
    arguments: [@Curl]
  Curl:
    class:     Curl
```

As we can see we're using a similar syntax than Symfony's DIC YAML files. Now, with our new library we can use the following code:

```php
use Pimple\Container;
use G\Yaml2Pimple\ContainerBuilder;
use G\Yaml2Pimple\YamlFileLoader;
use Symfony\Component\Config\FileLocator;
 
$container = new Container();
 
$builder = new ContainerBuilder($container);
$locator = new FileLocator(__DIR__);
$loader = new YamlFileLoader($builder, $locator);
$loader->load('services.yml');
 
$app = $container['App'];
echo $app->hello();
```

Now our Pimple/Container is just a Pimple/Container nothing more. It doesn't know anything about yaml, parsers and thing like that. It's doesn't have any extra responsibility. The responsibility of the parser falls on YamlFileLoader You can see the library in my [github](https://github.com/gonzalo123/yml2pimple) account. It's but one usage example of Symfony's Config component. It only allows Yaml files, but it can be extended with Xml files adding a XmlFileLoader.
