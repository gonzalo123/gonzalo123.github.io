---
layout: post
title: "How to run a Web Server from a PHP application"
date: "2013-11-11"
tags: 
  - "php"
  - "symfony"
  - "symfony2"
  - "technology"
  - "web-development"
  - "webdev"
  - "reactor-pattern"
  - "silex"
  - "web-server"
---

Normally we deploy our PHP applications in a webserver (such as apache, nginx, ...). I used to have one apache webserver in my personal computer to play with my applications, but from time to now I preffer to use PHP's built-in webserver for my experiments. It's really simple. Just run:

```commandline
php -S 0.0.0.0:8080 
``` 

and we've got one PHP webserver at our current directory. With another languages (such as node.js, Python) we can start a Web Server from our application. For example with node.js:

```javascript
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(8080, '0.0.0.0');
console.log('Server running at http://0.0.0.0:8080');
```

With PHP we cannot do it. Sure? That assertion isn't really true. We can do it. I've just create one small library to do it in two different ways. First running the built-in web server and also running one [React](http://reactphp.org/) web server.

I want to share the same interface to start the server. In this implementation we will register one callback to handle incomming requests. This callback will accept a Symfony\Component\HttpFoundation\Request and it will return a Symfony\Component\HttpFoundation\Response. Then we will start our server listening to one port and we will run our callback per Request (a simple implementeation of the [reactor](http://en.wikipedia.org/wiki/Reactor_pattern) pattern)

We will create a static factory to create the server 

```php
namespace G\HttpServer;
use React;
 
class Builder
{
    public static function createBuiltInServer($requestHandler)
    {
        $server = new BuiltInServer();
        $server->registerHandler($requestHandler);
 
        return $server;
    }
 
    public static function createReactServer($requestHandler)
    {
        $loop   = React\EventLoop\Factory::create();
        $socket = new React\Socket\Server($loop);
 
        $server = new ReactServer($loop, $socket);
        $server->registerHandler($requestHandler);
 
        return $server;
    }
}
```

Each server ([BuiltIn](https://github.com/gonzalo123/httpserver/blob/master/lib/G/HttpServer/BuiltInServer.php), and [React](https://github.com/gonzalo123/httpserver/blob/master/lib/G/HttpServer/ReactServer.php)) has its own implementation.

And basically that's all. We can run a simple webserver with the built-in server 

```php
use G\HttpServer\Builder;
use Symfony\Component\HttpFoundation\Request;
 
Builder::createBuiltInServer(function (Request $request) {
        return "Hello " . $request->get('name');
    })->listen(1337);
```

Or the same thing but with React 

```php
use G\HttpServer\Builder;
use Symfony\Component\HttpFoundation\Request;
 
Builder::createReactServer(function (Request $request) {
        return "Hello " . $request->get('name');
    })->listen(1337);
```

As you can see our callback handles one Request and returns one Response (The typical HttpKernel), because of that we also can run one Silex application: With built-in: 

```php
use G\HttpServer\Builder;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Silex\Application();
 
$app->get('/', function () {
        return 'Hello';
    });
 
$app->get('/hello/{name}', function ($name) {
        return 'Hello ' . $name;
    });
 
Builder::createBuiltInServer(function (Request $request) use ($app) {
        return $app->handle($request);
    })->listen(1337);
```

And the same with React: 

```php
use G\HttpServer\Builder;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Silex\Application();
 
$app->get('/', function () {
        return 'Hello';
    });
 
$app->get('/hello/{name}', function ($name) {
        return 'Hello ' . $name;
    });
 
Builder::createReactServer(function (Request $request) use ($app) {
        return $app->handle($request);
    })->listen(1337);
```

As an exercise I also have created one small benchmark (with both implementations) with [apache ab](http://httpd.apache.org/docs/2.2/programs/ab.html) running 100 request with a 10 request at the same time. Here you can see the outcomes.

<table border="0" cellspacing="0"><tbody><tr><td align="LEFT" bgcolor="#00FFFF" height="16"><b>&nbsp;</b></td><td align="CENTER" bgcolor="#00FFFF"><b>builtin</b></td><td align="CENTER" bgcolor="#00FFFF"><b>react</b></td></tr><tr><td align="LEFT" bgcolor="#FFFF00" height="16"><b>Simple response</b></td><td align="CENTER" bgcolor="#FFFF00"><b>&nbsp;</b></td><td align="CENTER" bgcolor="#FFFF00"><b>&nbsp;</b></td></tr><tr><td colspan="3" align="CENTER" valign="MIDDLE" bgcolor="#C0C0C0" height="16"><b>ab -n 100 -c 10 http://localhost:1337/</b></td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time taken for tests</td><td align="CENTER">0.878 seconds</td><td align="CENTER">0.101 seconds</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Requests per second (mean)</td><td align="CENTER">113.91 [#/sec]</td><td align="CENTER">989.33 [#/sec]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request (mean)</td><td align="CENTER">87.791 [ms]</td><td align="CENTER">10.108 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request (mean across all concurrent requests)</td><td align="CENTER">8.779 [ms]</td><td align="CENTER">1.011 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Transfer rate</td><td align="CENTER">21.02 [Kbytes/sec]</td><td align="CENTER">112.07 [Kbytes/sec]</td></tr><tr><td align="LEFT" bgcolor="#FFFF00" height="16"><b>Silex application</b></td><td align="LEFT" bgcolor="#FFFF00"></td><td align="LEFT" bgcolor="#FFFF00"></td></tr><tr><td colspan="3" align="CENTER" valign="MIDDLE" bgcolor="#C0C0C0" height="16"><b>ab -n 100 -c 10 http://localhost:1337/</b></td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time taken for tests</td><td align="CENTER">2.241 seconds</td><td align="CENTER">0.247 seconds</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Requests per second (mean)</td><td align="CENTER">44.62 [#/sec]</td><td align="CENTER">405.29 [#/sec]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request</td><td align="CENTER">224.119 [ms]</td><td align="CENTER">24.674 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request (mean across all concurrent requests)</td><td align="CENTER">22.412 [ms]</td><td align="CENTER">2.467 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Transfer rate</td><td align="CENTER">10.89 [Kbytes/sec]</td><td align="CENTER">75.60 [Kbytes/sec]</td></tr><tr><td colspan="3" align="CENTER" valign="MIDDLE" bgcolor="#C0C0C0" height="16"><b>ab -n 100 -c 10 http://localhost:1337/hello/gonzalo</b></td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time taken for tests</td><td align="CENTER">2.183 seconds</td><td align="CENTER">0.271 seconds</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Requests per second (mean)</td><td align="CENTER">45.81 [#/sec] (mean)</td><td align="CENTER">369.67 [#/sec]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request (mean)</td><td align="CENTER">218.290 [ms] (mean)</td><td align="CENTER">27.051 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Time per request (mean across all concurrent requests)</td><td align="CENTER">21.829 [ms]</td><td align="CENTER">2.705 [ms]</td></tr><tr><td align="LEFT" bgcolor="#C0C0C0" height="16">Transfer rate</td><td align="CENTER">11.54 [Kbytes/sec]</td><td align="CENTER">71.84 [Kbytes/sec]</td></tr></tbody></table>

Built-in web server is not suitable for production environments, but React would be a useful tool in some cases (maybe not good for running Facebook but good enough for punctual situations).

Library is available at [github](https://github.com/gonzalo123/httpserver) and also you can use it with [composer](https://packagist.org/packages/gonzalo123/httpserver)
