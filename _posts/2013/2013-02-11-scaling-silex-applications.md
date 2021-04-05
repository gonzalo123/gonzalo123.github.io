---
layout: post
title: "Scaling Silex applications"
date: "2013-02-11"
tags:
  - "dependency-injection"
  - "php"
  - "symfony"
  - "technology"
  - "dic"
  - "language-php"
  - "silex"
---

In my humble opinion [Silex](http://silex.sensiolabs.org/) is great. It's perfect to create prototypes, but when our application grows up it turns into a mess. That was what I thought until the last month, when I attended to a great [talk](http://www.slideshare.net/javier.eguiluz/silex-desarrollo-web-gil-y-profesional-con-php) about Silex with Javier Eguiluz. OK. Scaling Silex it's not the same than with a Symfony application, but it's possible.

It's pretty straightforward to create a Silex application with composer:

```json
{
    "require": {
        "silex/silex": "1.0.*"
    },
    "minimum-stability": "dev"
}
```

But there's a better way. We can use the Fabien Potencier's [skeleton](https://github.com/fabpot/Silex-Skeleton). With this skeleton we can organize our code better.

We also can use classes as controllers instead of using a closure with all the code. Igor Wiedler has a great post about this. You can read it [here](https://igor.io/2012/11/09/scaling-silex.html).

Today I'm playing with Silex and I want to show you something. Let's start:

Probably you know that I'm a big fan of Symfony's Dependency Injection Container (you can read about it [here](http://gonzalo123.com/2012/09/03/dependency-injection-containers-with-php-when-pimple-is-not-enough/) and [here](http://gonzalo123.com/2013/01/07/handling-several-pdo-database-connections-in-symfony2-through-the-dependency-injection-container-with-php/)), but Silex uses Pimple. In fact the Silex application extends Pimple Class. My idea is the following one:

In the Igor's post we can see how to use things like that:

```php
$app->match('/video/{id}', 'Gonzalo123\ApiController::indexAction')->method('GET')->bind('video_info');
```

My idea is to store this information within a Service Container (we will use Symfony's DIC). For example here we can see our routes.yml:

```yaml
routes:
  video_info:
    pattern:  /video/{id}
    controller: Gonzalo123\ApiController::initAction
    requirements:
      _method:  GET
```

As we can see we need to implement one Extension for the alias "routes". We only will implement the needed functions for YAML files in this example.

```php
<?php
 
use Symfony\Component\DependencyInjection\Extension\ExtensionInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
 
class SilexRouteExtension implements ExtensionInterface
{
    /**
     * Loads a specific configuration.
     *
     * @param array            $config    An array of configuration values
     * @param ContainerBuilder $container A ContainerBuilder instance
     *
     * @throws InvalidArgumentException When provided tag is not defined in this extension
     *
     * @api
     */
    public function load(array $config, ContainerBuilder $container)
    {
 
    }
 
    /**
     * Returns the namespace to be used for this extension (XML namespace).
     *
     * @return string The XML namespace
     *
     * @api
     */
    public function getNamespace()
    {
 
    }
 
    /**
     * Returns the base path for the XSD files.
     *
     * @return string The XSD base path
     *
     * @api
     */
    public function getXsdValidationBasePath()
    {
 
    }
 
    /**
     * Returns the recommended alias to use in XML.
     *
     * This alias is also the mandatory prefix to use when using YAML.
     *
     * @return string The alias
     *
     * @api
     */
    public function getAlias()
    {
        return "routes";
 
    }
}
```

And now we only need to prepare the DIC. According to Fabien's recommendation in his Silex skeleton, we only need to change the src/controllers.php

```php
<?php
 
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
 
// Set up container
$container = new ContainerBuilder();
$container->registerExtension(new SilexRouteExtension);
$loader = new YamlFileLoader($container, new FileLocator(__DIR__ . '/../config/'));
// load configuration
$loader->load('routes.yml');
$app['container'] = $container;
 
$app->mount('/api', include 'controllers/myApp.php');
 
$container->compile();
 
$app->error(function (\Exception $e, $code) use ($app) {
    if ($app['debug']) {
        return;
    }
 
    $page = 404 == $code ? '404.html' : '500.html';
 
    return new Response($app['twig']->render($page, array('code' => $code)), $code);
});
```

and now we define the config/routes.yml

```yaml
routes:
  video_info:
    pattern:  /video/{videoId}
    controller: Gonzalo123\ApiController::initAction
    requirements:
```

And finally the magic in our controllers/myApp.php:

```php
<?php
 
$myApp = $app['controllers_factory'];
 
foreach ($container->getExtensionConfig('routes')[0] as $name => $route) {
    $controller = $myApp->match($route['pattern'], $route['controller']);
    $controller->method($route['requirements']['_method']);
    $controller->bind($name);
}
return $myApp;
```

The class for this example is: src/Gonzalo123/ApiController.php

```php
<?php
 
namespace Gonzalo123;
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
 
use Symfony\Component\HttpFoundation\JsonResponse;
 
class ApiController
{
    public function initAction(Request $request, Application $app)
    {
        return new JsonResponse(array(1, 1, $request->get('id')));
    }
}
```

As you can see the idea is to use classes as controllers, define them within the service container and build the silex needed code iterating over the configuration. What do you think?
