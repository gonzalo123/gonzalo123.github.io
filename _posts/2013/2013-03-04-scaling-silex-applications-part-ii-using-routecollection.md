---
layout: post
title: "Scaling Silex applications (part II). Using RouteCollection"
date: "2013-03-04"
tags: 
  - "php"
  - "symfony"
  - "technology"
  - "silex"
---

In the post [Scaling Silex applications](http://gonzalo123.com/2013/02/11/scaling-silex-applications/ "Scaling Silex applications") I wanted to organize a one Silex application. In one [comment](http://gonzalo123.com/2013/02/11/scaling-silex-applications/#comment-3834) Igor Wiedler recommended us to use RouteCollections instead of define the routes with a Symfony's Dependency Injection Container. Because of that I started to hack a little bit about it and here I show you my outcomes:

I want to build an imaginary application with silex. This application has also one Api and one little blog. I want to organize those parts. Our index.php file

```php
<?php
// www/index.php
require_once __DIR__ . '/../vendor/autoload.php';
 
use Symfony\Component\Config\FileLocator;
use Symfony\Component\Routing\Loader\YamlFileLoader;
use Symfony\Component\Routing\RouteCollection;
use Silex\Application;
 
$app = new Application();
 
$app['routes'] = $app->extend('routes', function (RouteCollection $routes, Application $app) {
    $loader     = new YamlFileLoader(new FileLocator(__DIR__ . '/../config'));
    $collection = $loader->load('routes.yml');
    $routes->addCollection($collection);
 
    return $routes;
});
 
$app->run();
```

Now our routes.yml file: 
```yaml
# config/routes.yml
 
home:
  path: /
  defaults: { _controller: 'Gonzalo123\AppController::homeAction' }
 
hello:
  path: /hello/{name}
  defaults: { _controller: 'Gonzalo123\AppController::helloAction' }
 
api:
  prefix: /api
  resource: api.yml
 
blog:
  prefix: /blog
  resource: blog.yml
```

As we can see we have separated the main routing file into different files: api.yml (for the Api) and blog.yml (for the blog)

```yaml
# config/api.yml
 
api.list:
  path:     /list
  defaults: { _controller: 'Gonzalo123\ApiController::listAction' }
```

```yaml
# blog.yml
 
blog.home:
  path:     /
  defaults: { _controller: 'Gonzalo123\BlogController::homeAction' }
```

And now we can create our controllers: 

```php
<?php
// lib/Gonzalo123/AppController.php
 
namespace Gonzalo123;
 
use Symfony\Component\HttpFoundation\Response;
use Silex\Application;
 
class AppController
{
    public function homeAction()
    {
        return new Response("AppController::homeAction");
    }
 
    public function helloAction(Application $app, $name)
    {
        return new Response("Hello" . $app->escape($name));
    }
}
```

```php
<?php
// lib/Gonzalo123/ApiController.php
 
namespace Gonzalo123;
 
use Symfony\Component\HttpFoundation\Response;
 
class ApiController
{
    public function listAction()
    {
        return new Response("AppController::listAction");
    }
}
```

```php
<?php
// lib/Gonzalo123/BlogController.php
 
namespace Gonzalo123;
 
use Symfony\Component\HttpFoundation\Response;
 
class BlogController
{
    public function homeAction()
    {
        return new Response("BlogController::homeAction");
    }
}
```

And that's all. Here also the needed dependencies within our composer.json file 

```json
{
    "require":{
        "silex/silex":"1.0.*@dev",
        "symfony/yaml":"v2.2.0",
        "symfony/config":"v2.2.0"
    },
    "autoload":{
        "psr-0":{
            "":"lib/"
        }
    }
}
```

source code at [github](https://github.com/gonzalo123/silexRouteCollection).
