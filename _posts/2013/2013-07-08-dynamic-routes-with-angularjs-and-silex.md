---
layout: post
title: "Dynamic routes with AngularJS and Silex"
date: "2013-07-08"
tags: 
  - "angularjs"
  - "silex"
  - "technology"
  - "language-php"
---

These days I'm playing with AngularJS. Today I want to experiment with dynamic routes. Let me show your an example. Imagine a simple route configuration:

```php
$routeProvider.when('/page/page1', {templateUrl: 'partials/page1.html', controller: Controller1});
$routeProvider.when('/page/page2', {templateUrl: 'partials/page2.html', controller: Controller2});
$routeProvider.when('/page/page3', {templateUrl: 'partials/page3.html', controller: Controller3});
```

It's very simple but: What happens if our application is big and it grows fast? We need to add new lines and reload the browser. With AngularJS we can add paramenters to the routes:

```javascript
$routeProvider.when('/page/:page', {templateUrl: 'partials/page.html', controller: Controller});
```

Now we don't need to add new routes but what can we do with the partials? After browse the web and stack overflow finally I found this solution:

```javascript
.when('/page/:page', {template: '<div ng-include="templateUrl">Loading...</div>', controller: DynamicController})
```

And now we need to define our DynamicController and load there our needed partial: 

```javascript
function DynamicController($scope, $routeParams) {
    var unique = (new Date()).getTime();
    $scope.templateUrl = '/api/pages/' + $routeParams.page + '?unique=' + unique;
}
```

We can load our partial as a static file but in this example I'm using a Silex backend to provide my partials.

```php
<?php
require_once __DIR__ . '/../../vendor/autoload.php';
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Response;
$app          = new Application();
$app['debug'] = true;
 
$app->register(new Silex\Provider\TwigServiceProvider(), [
    'twig.path'    =>  __DIR__.'/../../views',
    'twig.options' => [
       'cache'       => __DIR__ . '/../../cache',
       'auto_reload' => true
    ]
]);
 
$app->before(function() use ($app){
    $app['twig']->setLexer( new Twig_Lexer($app['twig'], [
        'tag_comment'   => ['[#', '#]'],
        'tag_block'     => ['[%', '%]'],
        'tag_variable'  => ['[[', ']]'],
        'interpolation' => ['#[', ']'],
    ]));
});
 
$app->get('/pages/{name}', function($name) use ($app){
    return $app['twig']->render('hello.html.twig', ['name' => $name]);
});
 
$app->get('/pages/js/{name}', function($name) use ($app){
    $response = new Response($app['twig']->render('hello.js', ['name' => $name]));
    $response->headers->set("Content-Type", 'application/javascript');
 
    return $response;
});
 
$app->run();
```

As you can see we need to use one small hack to use twig and AngularJS together. They aren't good friends with the default configuration. Both uses the same syntax '{{ expresion }}'. Because of that we will change Twig's default behaviour within a middleware.

And basically that's all. You can see a working example in my [github account](https://github.com/gonzalo123/angularRemoteRoutes) using [Booststrap](http://twitter.github.io/bootstrap/) and [UI Booststrap](http://angular-ui.github.io/bootstrap/)
