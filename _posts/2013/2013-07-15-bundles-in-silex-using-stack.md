---
layout: post
title: "Bundles in Silex using Stack"
date: "2013-07-15"
tags: 
  - "php"
  - "symfony"
  - "symfony2"
  - "technology"
  - "silex"
  - "stack"
  - "technology"
---

In the last [Desymfony](http://desymfony.com/) conference I was speaking with [Luis Cordova](http://www.craftitonline.com/) and he introduced me ["Stack"](http://stackphp.com) (I must admit Stack was in my to-study-list but only marked as favorite). The idea behind Stack is really cool. (In fact every project where Igor Wiedler appears is brilliant, even the chicken [one](https://github.com/igorw/chicken-php) :)).

Nowadays almost every modern framework/applications implements [HttpKernelInterface](https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpKernel/HttpKernelInterface.php) (Symfony, Laravel, Drupal, Silex, Yolo and even the framework that I'm working in ;)) and we can build complex applications mixing different components and decorate our applications with an elegant syntax.

The first thing than come to my mind after studying Stack is to join different Silex applications in a similar way than Symfony (the full stack framework) uses bundles. And the best part of this idea is that it's pretty straightforward. Let me show you one example:

Imagine that we're working with one application with a blog and one API. In this case our blog and our API are Silex applications (but they can be one Symfony application and one Silex application for example).

That's our API application: 
```php
use Silex\Application;
 
$app = new Application();
$app->get('/', function () {
        return "Hello from API";
    });
 
$app->run();
```

And here our blog application: 

```php
use Silex\Application;
 
$app = new Application();
$app->get('/', function () {
        return "Hello from Blog";
    });
 
$app->run();
```

We can organize our application using [mounted](http://silex.sensiolabs.org/doc/organizing_controllers.html) controllers or even using [RouteCollections](http://gonzalo123.com/2013/03/04/scaling-silex-applications-part-ii-using-routecollection/) but today we're going to use Stack and it's cool [url-map](https://github.com/stackphp/url-map).

First we are going to create our base application. To do this we're going to implement the simplest Kernel in the world, that's answers with "Hello" to every request: 

```php
use Symfony\Component\HttpKernel\HttpKernelInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
 
class MyKernel implements HttpKernelInterface
{
    public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
    {
        return new Response("Hello");
    }
}
```

Stack needs HttpKernelInterface and Silex\Application implements this interface, so we can change our Silex applications to return the instance instead to run the application: 

```php
// app/api.php
use Silex\Application;
 
$app = new Application();
$app->get('/', function () {
        return "Hello from API";
    });
 
return $app;
```

```php
// app/blog.php
use Silex\Application;
 
$app = new Application();
$app->get('/', function () {
        return "Hello from API";
    });
 
return $app;
```

And now we will attach those two Silex applications to our Kernel: 

```php
use Symfony\Component\HttpFoundation\Request;
 
$app = (new Stack\Builder())
    ->push('Stack\UrlMap', [
            "/blog" => include __DIR__ . '/app/blog.php',
            "/api" => include __DIR__ . '/app/api.php'
        ])->resolve(new MyKernel());
 
$request = Request::createFromGlobals();
 
$response = $app->handle($request);
$response->send();
 
$app->terminate($request, $response);
```

And that's all. I don't know what you think but with Stack one big window just opened in my mind. Cool, isn't it?

You can see this working example in my [github](https://github.com/gonzalo123/silexBundles)
