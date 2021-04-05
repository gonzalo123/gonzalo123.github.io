---
layout: post
title: "POST Request logger using websockets"
date: "2015-11-16"
tags: 
  - "gps"
  - "silex"
  - "cordova"
  - "phonegap"
  - "php"
  - "technology"
---

Last days I've been working with background geolocation with an ionic application. There's a cool [plugin](https://github.com/transistorsoft/cordova-background-geolocation-lt) to do that. The free version of the plugin works fine. But there's a also a premium version with improvements, especially in battery consumption with Android devices.

Basically this plugin performs a POST request to the server with the GPS data. When I was developing my application I needed a simple HTTP server to see the POST requests. Later I'll code the backend to handle those requests. I can develop a simple Silex application with a POST route and log the request in a file or flush those request to the console. This'd have been easy but as far as I'm a big fan of WebSockets (yes I must admit that I want to use WebSockets everywere :) I had one idea in my mind. The idea was create a simple HTTP server to handle my GPS POST requests but instead of logging the request I will emit a WebSocket. Then I can create one site that connects to the WebSocket server and register on screen the POST request. Ok today I'm a bit lazy to fight with the Frontend so my log will be on the browser's console.

To build the application I'll reuse one of my projects in github: The [PHP dumper](http://gonzalo123.com/2015/05/11/php-dumper-using-websockets/). The idea is almost the same. I'll create a simple HTTP server with Silex with two routes. One to handle POST requests (the GPS ones) and another GET to allow me to connect to the WebSocket

That's the server. Silex, a bit of Twig, another bit of Guzzle and that's all

```php
use GuzzleHttp\Client;
use Silex\Application;
use Silex\Provider\TwigServiceProvider;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application([
    'debug'       => true,
    'ioServer'    => '//localhost:8888',
    'wsConnector' => 'http://127.0.0.1:26300'
]);
 
$app->register(new TwigServiceProvider(), [
    'twig.path' => __DIR__ . '/../views',
]);
 
$app['http.client'] = new Client();
 
$app->get("/{channel}", function (Application $app, $channel) {
    return $app['twig']->render('index.twig', [
        'channel'  => $channel,
        'ioServer' => $app['ioServer']
    ]);
});
 
$app->post("/{channel}", function (Application $app, $channel, Request $request) {
    $app['http.client']->get($app['wsConnector'] . "/info/{$channel}/" . json_encode($request->getContent()));
 
    return $app->json('OK');
});
 
$app->run();
```

That's the Twig template. Nothing especial: A bit of Bootstrap and one socket.io client. Each time user access to one "channel"'s url (GET /mychannel). It connects to websocket server

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

And each time background geolocation plugin POSTs GPS data Silex POST route will emit a WebSocket to the desired channel. Our WebSocket client just logs the GPS data using console.log. Is hard to explain but very simple process.

We also can emulate POST requests with this simple node script:

```javascript
var request = require('request');
 
request.post('http://localhost:8080/Hello', {form: {key: 'value'}}, function (error, response, body) {
    if (!error && response.statusCode == 200) {
        console.log(body)
    }
});
```

And that's all. You can see the whole code within my [github](https://github.com/gonzalo123/POSTlogger) account.
