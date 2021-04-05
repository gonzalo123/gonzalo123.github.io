---
layout: post
title: "Yet Another example of WebSockets, socket.io and AngularJs working with a Silex backend"
date: "2014-10-27"
tags: 
  - "angularjs"
  - "node-js"
  - "socket-io"
  - "express"
  - "html5"
  - "js"
  - "php"
  - "silex"
  - "symfony"
  - "websockets"
---

Remember my [last post about WebSockets](http://gonzalo123.com/2014/08/25/playing-with-websockets-angularjs-and-socket-io/ "Playing with websockets, angularjs and socket.io") and AngularJs? Today we're going to play with something similar. I want to create a key-value interface to play with websockets. Let me explain it a little bit.

First we're going to see the backend. One Silex application with two routes: a get one and a post one:

```php
<?php
 
include __DIR__ . '/../../vendor/autoload.php';
include __DIR__ . '/SqlLiteStorage.php';
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Silex\Provider\DoctrineServiceProvider;
 
$app = new Application([
    'debug'      => true,
    'ioServer'   => 'http://localhost:3000',
    'httpServer' => 'http://localhost:3001',
]);
 
$app->after(function (Request $request, Response $response) {
    $response->headers->set('Access-Control-Allow-Origin', '*');
});
 
$app->register(new G\Io\EmitterServiceProvider($app['httpServer']));
$app->register(new DoctrineServiceProvider(), [
    'db.options' => [
        'driver' => 'pdo_sqlite',
        'path'   => __DIR__ . '/../../db/app.db.sqlite',
    ],
]);
$app->register(new G\Io\Storage\Provider(new SqlLiteStorage($app['db'])));
 
$app->get('conf', function (Application $app, Request $request) {
    $chanel = $request->get('token');
    return $app->json([
        'ioServer' => $app['ioServer'],
        'chanel'   => $chanel
    ]);
});
 
$app->get('/{key}', function (Application $app, $key) {
    return $app->json($app['gdb.get']($key));
});
 
$app->post('/{key}', function (Application $app, Request $request, $key) {
    $content = json_decode($request->getContent(), true);
 
    $chanel = $content['token'];
    $app->json($app['gdb.post']($key, $content['value']));
 
    $app['io.emit']($chanel, [
        'key'   => $key,
        'value' => $content['value']
    ]);
 
    return $app->json(true);
});
 
$app->run();
```

As we can see we register one service provider:

```php
$app->register(new G\Io\Storage\Provider(new SqlLiteStorage($app['db'])));
```

This provider needs an instance of StorageIface

```php
namespace G\Io\Storage;
 
interface StorageIface
{
    public function get($key);
 
    public function post($key, $value);
}
```

Our implementation uses SqlLite, but it's pretty straightforward to change to another Database Storage or even a NoSql Database.

```php
use Doctrine\DBAL\Connection;
use G\Io\Storage\StorageIface;
 
class SqlLiteStorage implements StorageIface
{
    private $db;
 
    public function __construct(Connection $db)
    {
        $this->db = $db;
    }
 
    public function get($key)
    {
        $statement = $this->db->executeQuery('select value from storage where key = :KEY', ['KEY' => $key]);
        $data      = $statement->fetchAll();
 
        return isset($data[0]['value']) ? $data[0]['value'] : null;
    }
 
    public function post($key, $value)
    {
        $this->db->beginTransaction();
 
        $statement = $this->db->executeQuery('select value from storage where key = :KEY', ['KEY' =>; $key]);
        $data      = $statement->fetchAll();
 
        if (count($data) > 0) {
            $this->db->update('storage', ['value' => $value], ['key' => $key]);
        } else {
            $this->db->insert('storage', ['key' => $key, 'value' => $value]);
        }
 
        $this->db->commit();
 
        return $value;
    }
}
```

We also register another Service provider:

```php
$app->register(new G\Io\EmitterServiceProvider($app['httpServer']));
```

This provider's responsibility is to notify to the websocket's server when anything changes within the storage:

```php
namespace G\Io;
 
use Pimple\Container;
use Pimple\ServiceProviderInterface;
 
class EmitterServiceProvider implements ServiceProviderInterface
{
    private $server;
 
    public function __construct($url)
    {
        $this->server = $url;
    }
 
    public function register(Container $app)
    {
        $app['io.emit'] = $app->protect(function ($chanel, $params) use ($app) {
            $s = curl_init();
            curl_setopt($s, CURLOPT_URL, '{$this->server}/emit/?' . http_build_query($params) . '&_chanel=' . $chanel);
            curl_setopt($s, CURLOPT_RETURNTRANSFER, true);
            $content = curl_exec($s);
            $status  = curl_getinfo($s, CURLINFO_HTTP_CODE);
            curl_close($s);
 
            if ($status != 200) throw new \Exception();
 
            return $content;
        });
    }
}
```

The Websocket server is a simple [socket.io](http://socket.io/) server as well as a [Express](http://expressjs.com/) server to handle the backend's triggers.

```javascript
var
    express = require('express'),
    expressApp = express(),
    server = require('http').Server(expressApp),
    io = require('socket.io')(server, {origins: 'localhost:*'})
    ;
 
expressApp.get('/emit', function (req, res) {
    io.sockets.emit(req.query._chanel, req.query);
    res.json('OK');
});
 
expressApp.listen(3001);
 
server.listen(3000);
```

Our client application is an AngularJs application:

```html
<!doctype html>
<html ng-app="app">
<head>
    <script src="//localhost:3000/socket.io/socket.io.js"></script>
    <script src="assets/angularjs/angular.js"></script>
    <script src="js/app.js"></script>
    <script src="js/gdb.js"></script>
</head>
<body>
  
<div ng-controller="MainController">
    <input type="text" ng-model="key">
    <button ng-click="change()">change</button>
</div>
  
</body>
</html>
angular.module('app', ['Gdb'])
 
    .run(function (Gdb) {
        Gdb.init({
            server: 'http://localhost:8080/gdb',
            token: '4b96716bcb3d42fc01ff421ea2cfd757'
        });
    })
 
    .controller('MainController', function ($scope, Gdb) {
        $scope.change = function () {
            Gdb.set('key', $scope.key).then(function() {
                console.log(&quot;Value set&quot;);
            });
        };
 
        Gdb.get('key').then(function (data) {
            $scope.key = data;
        });
 
        Gdb.watch('key', function (value) {
            console.log(&quot;Value updated&quot;);
            $scope.key = value;
        });
    })
;
```

As we can see the AngularJs application uses one small library called Gdb to handle the communications with the backend and WebSockets:

```javascript
angular.module('Gdb', [])
    .factory('Gdb', function ($http, $q, $rootScope) {
 
        var socket,
            gdbServer,
            token,
            watches = {};
 
        var Gdb = {
            init: function (conf) {
                gdbServer = conf.server;
                token = conf.token;
 
                $http.get(gdbServer + '/conf', {params: {token: token}}).success(function (data) {
                    socket = io.connect(data.ioServer);
                    socket.on(data.chanel, function (data) {
                        watches.hasOwnProperty(data.key) ? watches[data.key](data.value) : null;
                        $rootScope.$apply();
                    });
                });
            },
 
            set: function (key, value) {
                var deferred = $q.defer();
 
                $http.post(gdbServer + '/' + key, {value: value, token: token}).success(function (data) {
                    deferred.resolve(data);
                });
 
                return deferred.promise;
            },
 
            get: function (key) {
                var deferred = $q.defer();
 
                $http.get(gdbServer + '/' + key, {params: {token: token}}).success(function (data) {
                    deferred.resolve(JSON.parse(data));
                });
 
                return deferred.promise;
            },
 
            watch: function (key, closure) {
                watches[key] = closure;
            }
        };
 
        return Gdb;
    });
```

And that's all. You can see the whole project at [github](https://github.com/gonzalo123/gdb).
