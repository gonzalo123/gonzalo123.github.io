---
layout: post
title: "PHP Dumper using Websockets"
date: "2015-05-11"
tags: 
  - "js"
  - "silex"
  - "socket-io"
  - "node-js"
  - "npm"
  - "php"
  - "technology"
  - "websockets"
---

Another crazy idea. I want to dump my backend output in the browser's console. There're several PHP dumpers. For example Raul Fraile's [LadyBug](https://github.com/raulfraile/Ladybug). There're also libraries to do exactly what I want to do, such as [Chrome Logger](https://craig.is/writing/chrome-logger). But I wanted to use Websockets and dump values in real time, without waiting to the end of backend script. Why? The answer is simple: Because I wanted to it :)

I've written [several](http://gonzalo123.com/category/technology/websockets/) post about Websockets, Silex, PHP. In this case I'll use a similar approach than the previous posts. First I've created a simple Webscocket server with socket.io. This server also starts a [Express](http://expressjs.com/) server to handle internal messages from the Silex Backend

```javascript
var CONF = {
        IO: {HOST: '0.0.0.0', PORT: 8888},
        EXPRESS: {HOST: '0.0.0.0', PORT: 26300}
    },
    express = require('express'),
    expressApp = express(),
    server = require('http').Server(expressApp),
    io = require('socket.io')(server, {origins: 'localhost:*'})
    ;
 
expressApp.get('/:type/:session/:message', function (req, res) {
    console.log(req.params);
    var session = req.params.session,
        type = req.params.type,
        message = req.params.message;
 
    io.sockets.emit('dumper.' + session, {title: type, data: JSON.parse(message)});
    res.json('OK');
});
 
io.sockets.on('connection', function (socket) {
    console.log("Socket connected!");
});
 
expressApp.listen(CONF.EXPRESS.PORT, CONF.EXPRESS.HOST, function () {
    console.log('Express started');
});
 
server.listen(CONF.IO.PORT, CONF.IO.HOST, function () {
    console.log('IO started');
});
```

Now we create a simple Service provider to connect our Silex Backend to our Express server (and send the dumper's messages using the websocket connection)

```php
<?php
 
namespace Dumper\Silex\Provider;
 
use Silex\Application;
use Silex\ServiceProviderInterface;
use Dumper\Dumper;
use Silex\Provider\SessionServiceProvider;
use GuzzleHttp\Client;
 
class DumperServiceProvider implements ServiceProviderInterface
{
    private $wsConnector;
    private $client;
 
    public function __construct(Client $client, $wsConnector)
    {
        $this->client = $client;
        $this->wsConnector = $wsConnector;
    }
 
    public function register(Application $app)
    {
        $app->register(new SessionServiceProvider());
 
        $app['dumper'] = function () use ($app) {
            return new Dumper($this->client, $this->wsConnector, $app['session']->get('uid'));
        };
 
        $app['dumper.init'] = $app->protect(function ($uid) use ($app) {
            $app['session']->set('uid', $uid);
        });
 
        $app['dumper.uid'] = function () use ($app) {
            return $app['session']->get('uid');
        };
    }
 
    public function boot(Application $app)
    {
    }
}
```

Finally our Silex Application looks like that: 

```php
include __DIR__ . '/../vendor/autoload.php';
 
use Silex\Application;
use Silex\Provider\TwigServiceProvider;
use Dumper\Silex\Provider\DumperServiceProvider;
use GuzzleHttp\Client;
 
$app = new Application([
    'debug' => true
]);
 
$app->register(new DumperServiceProvider(new Client(), 'http://192.168.1.104:26300'));
 
$app->register(new TwigServiceProvider(), [
    'twig.path' => __DIR__ . '/../views',
]);
 
$app->get("/", function (Application $app) {
    $uid = uniqid();
 
    $app['dumper.init']($uid);
 
    return $app['twig']->render('index.twig', [
        'uid' => $uid
    ]);
});
 
$app->get('/api/hello', function (Application $app) {
    $app['dumper']->error("Hello world1");
    $app['dumper']->info([1,2,3]);
 
    return $app->json('OK');
});
 
 
$app->run();
```

In the client side we have one index.html. I've created Twig template to pass uid to the dumper object (the websocket channel to listen to), but we also can fetch this uid from the backend with one ajax call.

```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>Dumper example</title>
</head>
<body>
 
<a href="#" onclick="api('hello')">hello</a>
 
<!-- We use jQuery just for the demo. Library doesn't need jQuery -->
<script src="assets/jquery/dist/jquery.min.js"></script>
<!-- We load the library -->
<script src="js/dumper.js"></script>
 
<script>
    dumper.startSocketIo('{{ uid }}', '//localhost:8888');
    function api(name) {
        // we perform a remote api ajax call that triggers websockets
        $.getJSON('/api/' + name, function (data) {
            // Doing nothing. We only call the api to test php dumper
        });
    }
</script>
</body>
</html>
```

I use jQuery to handle ajax request and to connect to the websocket dumper object (it doesn't deppend on jQuery, only depend on socket.io)

```javascript
var dumper = (function () {
    var socket, sessionUid, socketUri, init;
 
    init = function () {
        if (typeof(io) === 'undefined') {
            setTimeout(init, 100);
        } else {
            socket = io(socketUri);
 
            socket.on('dumper.' + sessionUid, function (data) {
                console.group('Dumper:', data.title);
                switch (data.title) {
                    case 'emergency':
                    case 'alert':
                    case 'critical':
                    case 'error':
                        console.error(data.data);
                        break;
                    case 'warning':
                        console.warn(data.data);
                        break;
                    case 'notice':
                    case 'info':
                    //case 'debug':
                        console.info(data.data);
                        break;
                    default:
                        console.log(data.data);
                }
                console.groupEnd();
            });
        }
    };
 
    return {
        startSocketIo: function (uid, uri) {
            var script = document.createElement('script');
            var node = document.getElementsByTagName('script')[0];
 
            sessionUid = uid;
            socketUri = uri;
            script.src = socketUri + '/socket.io/socket.io.js';
            node.parentNode.insertBefore(script, node);
 
            init();
        }
    };
})();
```

[![youtube](https://img.youtube.com/vi/74J-eO187OQ/0.jpg)](https://www.youtube.com/watch?v=74J-eO187OQ)

Source code is available in my [github](https://github.com/gonzalo123/dumper) account
