---
layout: post
title: "Playing with websockets, angularjs and socket.io"
date: "2014-08-25"
tags: 
  - "angularjs"
  - "node-js"
  - "php"
  - "socket-io"
  - "technology"
  - "js"
  - "silex"
  - "websockets"
---

I'm a big fan of websockets. I've got various post about them ([here](http://gonzalo123.com/2013/12/24/integrating-websockets-with-php-applications-silex-and-socket-io-playing-together/ "Integrating WebSockets with PHP applications. Silex and socket.io playing together."), [here](http://gonzalo123.com/2011/05/09/real-time-monitoring-php-applications-with-websockets-and-node-js/ "Real time monitoring PHP applications with websockets and node.js")). Last months I'm working with [angularjs](https://angularjs.org/) projects and because of that I wanna play a little bit with websockets (with [socket.io](http://socket.io/)) and angularjs.

I want to build one angular service. 

```javascript
angular.module('io.service', []).
    factory('io', function ($http) {
        var socket,
            apiServer,
            ioEvent,
            watches = {};
 
        return {
            init: function (conf) {
                apiServer = conf.apiServer;
                ioEvent = conf.ioEvent;
 
                socket = io.connect(conf.ioServer);
                socket.on(ioEvent, function (data) {
                    return watches.hasOwnProperty(data.item) ? watches[data.item](data) : null;
                });
            },
 
            emit: function (arguments) {
                return $http.get(apiServer + '/request', {params: arguments});
            },
 
            watch: function (item, closure) {
                watches[item] = closure;
            },
 
            unWatch: function (item) {
                delete watches[item];
            }
        };
    });
```

And now we can build the application 
```javascript
angular.module('myApp', ['io.service']).
 
    run(function (io) {
        io.init({
            ioServer: 'http://localhost:3000',
            apiServer: 'http://localhost:8080/api',
            ioEvent: 'io.response'
        });
    }).
 
    controller('MainController', function ($scope, io) {
        $scope.$watch('question', function (newValue, oldValue) {
            if (newValue != oldValue) {
                io.emit({item: 'question', newValue: newValue, oldValue: oldValue});
            }
        });
 
        io.watch('answer', function (data) {
            $scope.answer = data.value;
            $scope.$apply();
        });
    });
```

And this's the html 
```html
<!doctype html>
<html>
 
<head>
    <title>ws experiment</title>
</head>
 
<body ng-app="myApp">
 
<div ng-controller="MainController">
 
    <input type="text" ng-model="question">
    <hr>
    <h1>Hello {{answer}}!</h1>
</div>
 
<script src="assets/angular/angular.min.js"></script>
<script src="//localhost:3000/socket.io/socket.io.js"></script>
 
<script src="js/io.js"></script>
<script src="js/app.js"></script>
 
</body>
</html>
```

The idea of the application is to watch one model's variable ('question' in this example) and each time it changes we will send the value to the websocket server and we'll so something (we will convert the string to upper case in our example)

As you can read in one of my previous post I don't like to send messages from the web browser to the websocket server directly (due do to authentication issues commented [here](http://gonzalo123.com/2013/12/24/integrating-websockets-with-php-applications-silex-and-socket-io-playing-together/ "Integrating WebSockets with PHP applications. Silex and socket.io playing together.")). I prefer to use one server (a Silex server in this example)

```php
include __DIR__ . '/../../vendor/autoload.php';
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application(['debug' => true]);
$app->register(new G\Io\EmitterServiceProvider());
 
$app->get('/request', function (Application $app, Request $request) {
 
    $params = [
        'item'     => $request->get('item'),
        'newValue' => strtoupper($request->get('newValue')),
        'oldValue' => $request->get('oldValue'),
    ];
 
    try {
        $app['io.emit']($params);
        $params['status'] = true;
    } catch (\Exception $e) {
        $params['status'] = false;
    }
 
    return $app->json($params);
});
 
$app->run();
```

You can see the code within my [github](https://github.com/gonzalo123/wsexperiment) account.

[![youtube](https://img.youtube.com/vi/8gTZ2kYTx8U/0.jpg)](https://www.youtube.com/watch?v=8gTZ2kYTx8U)

