---
layout: post
title: "Working with AngularJS and Silex as Resource provider"
date: "2013-06-03"
tags: 
  - "angularjs"
  - "php"
  - "symfony"
  - "symfony2"
  - "technology"
  - "rest"
  - "silex"
---

This days I'm playing with [AngularJS](http://angularjs.org/). Angular is a great framework when we're building complex front-end applications with JavaScript. And the best part is that it's very simple to understand (and I like simple things indeed). Today we are going to play with [Resources](http://docs.angularjs.org/api/ngResource.$resource). Resources are great when we need to use RestFull resources from the server. In this example we're going to use [Silex](http://silex.sensiolabs.org/) in the backend. Let's start.

First of all we must realize that resources aren't included in the main AngularJS js file and we need to include angular-resource.js (it comes with Angular package). We don't really need resources. We can create our http services with AngularJS without using this extra js file but it provides a very clean abstraction (at least for me) and I like it.

```html
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.7/angular.min.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.7/angular-resource.min.js"></script>
```

We're going to create a simple application with CRUD operations in the table. In the example we will use one simple SqlLite database (included in the github repository)

```postgresql
CREATE TABLE main.messages (
  id INTEGER PRIMARY KEY  NOT NULL ,
  author VARCHAR NOT NULL ,
  message VARCHAR NOT NULL );
```

Our main (and only one) html file:

```html
<!DOCTYPE html>
<html ng-app="MessageService">
<head>
    <title>Angular Resource Example</title>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.7/angular.min.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.7/angular-resource.min.js"></script>
    <script src="js/services.js"></script>
    <script src="js/controllers.js"></script>
</head>
<body ng-controller="MessageController">
 
<h2>Message list <a href="#" ng-click="refresh()">refresh</a></h2>
<ul>
    <li ng-repeat="message in messages">
        <a href="#" ng-click="deleteMessage($index, message.id)">delete</a>
        #{{message.id}} - {{message.author}} - {{message.message}}
        <a href="#" ng-click="selectMessage($index)">edit</a>
    </li>
</ul>
 
<form>
    <input type="text" ng-model="author" placeholder="author">
    <input type="text" ng-model="message" placeholder="message">
 
    <button ng-click="add()" ng-show='addMode'>Create</button>
    <button ng-click="update()" ng-hide='addMode'>Update</button>
    <button ng-click="cancel()" ng-hide='addMode'>Cancel</button>
</form>
</body>
</html>
```

As we can see we will use ng-app=”MessageService” defined within the js/services.js file:

```javascript
angular.module('MessageService', ['ngResource']).factory('Message', ['$resource', function ($resource) {
    return $resource('/api/message/resource/:id');
}]);
```

And our controller in js/controllers.js:

```javascript
function MessageController($scope, Message) {
 
    var currentResource;
    var resetForm = function () {
        $scope.addMode = true;
        $scope.author = undefined;
        $scope.message = undefined;
        $scope.selectedIndex = undefined;
    }
 
    $scope.messages = Message.query();
    $scope.addMode = true;
 
    $scope.add = function () {
        var key = {};
        var value = {author: $scope.author, message: $scope.message}
 
        Message.save(key, value, function (data) {
            $scope.messages.push(data);
            resetForm();
        });
    };
 
    $scope.update = function () {
        var key = {id: currentResource.id};
        var value = {author: $scope.author, message: $scope.message}
        Message.save(key, value, function (data) {
            currentResource.author = data.author;
            currentResource.message = data.message;
            resetForm();
        });
    }
 
    $scope.refresh = function () {
        $scope.messages = Message.query();
        resetForm();
    };
 
    $scope.deleteMessage = function (index, id) {
        Message.delete({id: id}, function () {
            $scope.messages.splice(index, 1);
            resetForm();
        });
    };
 
    $scope.selectMessage = function (index) {
        currentResource = $scope.messages[index];
        $scope.addMode = false;
        $scope.author = currentResource.author;
        $scope.message = currentResource.message;
    }
 
    $scope.cancel = function () {
        resetForm();
    }
}
```

Now the backend part. As we said before we will use Silex. We're going to use also RouteCollections to define our routes (you can read about it [here](http://gonzalo123.com/2013/03/04/scaling-silex-applications-part-ii-using-routecollection/)). So our Silex application will be:

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';
 
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\Routing\Loader\YamlFileLoader;
use Symfony\Component\Routing\RouteCollection;
use Silex\Application;
 
$app = new Silex\Application();
 
$app['routes'] = $app->extend('routes', function (RouteCollection $routes, Application $app) {
        $loader     = new YamlFileLoader(new FileLocator(__DIR__ . '/config'));
        $collection = $loader->load('routes.yml');
        $routes->addCollection($collection);
 
        return $routes;
    }
);
 
$app->register(
    new Silex\Provider\DoctrineServiceProvider(),
    array(
        'db.options' => array(
            'driver' => 'pdo_sqlite',
            'path'   => __DIR__ . '/db/app.db.sqlite',
        ),
    )
);
 
$app->run();
```

We define our routes in the messageResource.yml

```yaml
getAll:
  path: /resource
  defaults: { _controller: 'Message\MessageController::getAllAction' }
  methods:  [GET]
 
getOne:
  path: /resource/{id}
  defaults: { _controller: 'Message\MessageController::getOneAction' }
  methods:  [GET]
 
deleteOne:
  path: /resource/{id}
  defaults: { _controller: 'Message\MessageController::deleteOneAction' }
  methods:  [DELETE]
 
addOne:
  path: /resource
  defaults: { _controller: 'Message\MessageController::addOneAction' }
  methods:  [POST]
 
editOne:
  path: /resource/{id}
  defaults: { _controller: 'Message\MessageController::editOneAction' }
  methods:  [POST]
```

And finally our Resource controller: 

```php
<?php
namespace Message;
 
use Silex\Application;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
 
class MessageController
{
    public function getAllAction(Application $app)
    {
        return new JsonResponse($app['db']->fetchAll("SELECT * FROM messages"));
    }
 
    public function getOneAction($id, Application $app)
    {
        return new JsonResponse($app['db']
            ->fetchAssoc("SELECT * FROM messages WHERE id=:ID", ['ID' => $id]));
    }
 
    public function deleteOneAction($id, Application $app)
    {
        return $app['db']->delete('messages', ['ID' => $id]);
    }
 
    public function addOneAction(Application $app, Request $request)
    {
        $payload = json_decode($request->getContent());;
 
        $newResource = [
            'id'      => (integer)$app['db']
                ->fetchColumn("SELECT max(id) FROM messages") + 1,
            'author'  => $payload->author,
            'message' => $payload->message,
        ];
        $app['db']->insert('messages', $newResource);
 
        return new JsonResponse($newResource);
    }
 
    public function editOneAction($id, Application $app, Request $request)
    {
        $payload = json_decode($request->getContent());;
        $resource = [
            'author'  => $payload->author,
            'message' => $payload->message,
        ];
        $app['db']->update('messages', $resource, ['id' => $id]);
 
        return new JsonResponse($resource);
    }
}
```

And that's all. Our prototype is working with AngularJS and Silex as REST provider. We must take care about one thing. Silex and AngularJS aren't agree in one thing about REST services. AngularJS removes the trailing slash in some cases. Silex (and Symfony) returns HTTP 302 moved temporaly when we're trying to access to the resource without the trailing slash but when we're working with mounted controllers we will obtain a 404 page not found (bug/feature?). That's because my REST service is /api/message/resource/:id instead of /api/message/:id. If I chose the second one, when angular tries to create a new resource, it will POST /api/message instead of POST /api/message/. We're using mounted routes in this example:

```yaml
messages:
  prefix: /message
  resource: messageResource.yml
```

With one simple Silex application (without mounted routes) in one file it doesn't happen (we will see HTTP 302 and a new request with the trailing slash). Because of that I use this small hack to bypass the problem.

You can see the full code of the example in my [github](https://github.com/gonzalo123/silexNgResourceExample) account

[![youtube](https://img.youtube.com/vi/5GaomWlL1xI\/0.jpg)](https://www.youtube.com/watch?v=5GaomWlL1xI\)
