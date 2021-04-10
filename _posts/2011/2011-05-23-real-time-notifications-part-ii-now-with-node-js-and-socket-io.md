---
layout: post
title: "Real time notifications (part II). Now with node.js and socket.io"
date: "2011-05-23"
tags: 
  - "html5"
  - "node-js"
  - "socket-io"
  - "technology"
  - "websockets"
---

In one of my previous posts I wrote about [Real time notifications with PHP](http://gonzalo123.wordpress.com/2011/03/14/real-time-notifications-with-php/ "Real time notifications with PHP"). I wanted to create a simple comet system fully written in PHP and JavaScript. It worked but as Scott Mattocks told me in a [comment](http://gonzalo123.wordpress.com/2011/03/14/real-time-notifications-with-php/#comment-1034) this implementation was still just doing short polling. The performance with this solution may be bad in a medium/hight traffic site. This days I’m playing with node.js and I want to create a simple test. I want to do exactly the same than the previous post but now with node.js instead of my PHP+js test. Let’s start

Now I want to use socket.io instead of pure web-sockets like my previous posts about node.js. For those who don’t know, [socket.io](http://socket.io/) is amazing library that allows us to use real-time technologies within every browsers (yes even with IE6). It uses one technology or another depending on the browser we are using, with the same interface for the developer. That's means if we're using Google Chrome we will use websockets, but if our browser does't support them, socket.io will choose another supported transports. Definitely socket.io is the jQuery of the websockets. The supporter transports are:

- WebSocket
- Adobe Flash Socket
- AJAX long polling
- AJAX multipart streaming
- Forever Iframe
- JSONP Polling

First we create our node.js server. A really simple one. 

```javascript
var http = require('http');
var io = require('socket.io');
 
server = http.createServer(function(req, res){
});
server.listen(8080);
 
// socket.io 
var socket = io.listen(server);
 
socket.on('connection', function(client){
  client.on('message', function(msg){
      socket.broadcast(msg);
  })
}); 
```

This server will broadcast the message received from the browser to all connected clients.

Our HTML page will look like that: \

```html
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>Comet Test</title>
    </head>
    <body>
        <p><a id='customAlert' href="#" onclick='socket.send("customAlert")'>publish customAlert</a></p>
        <p><a id='customAlert2' href="#" onclick='socket.send("customAlert2")'>publish customAlert2</a></p>
        <script src="http://localhost:8080/socket.io/socket.io.js" type="text/javascript"></script>
        <script type="text/javascript">
 
// Start the socket
var socket = new io.Socket(null, {port: 8080});
socket.connect();
 
socket.on('message', function(msg){
    console.log(msg);
});
        </script>
    </body>
</html>
```

As we can see we are including the js script called socket.io/socket.io.js. This script is served by our node server.

In fact we can use our node.js to serve everything (HTML, js, CSS) but in our example we will use only node.js for real-time stuff. Apache will serve the rest of the code (only a HTML file in this case).

And that’s all. Those few lines perform the same thing than our PHP and js code in the [other post’s](http://gonzalo123.wordpress.com/2011/03/14/real-time-notifications-with-php/) example. Our node.js implementation is definitely smarter than the PHP one. The socket.io library also allows us to use the example with all browser. Same code and without any browser nightmare (just like jQuery when we work with DOM).

Here I have a little screencast with the working example. As we will see there We will connect to the node server with firefox and chrome. Firefox will use xhr multipart and Chrome will use Websokets.

[![youtube](https://img.youtube.com/vi/vwy7qYeZvG8\/0.jpg)](https://www.youtube.com/watch?v=vwy7qYeZvG8\)

Another important issue of socket.io library is that we forget about the reconnection to the web-socket server, if something wrong happens (as we can see in [Real time monitoring PHP applications with web-sockets and node.js](http://gonzalo123.wordpress.com/2011/05/09/real-time-monitoring-php-applications-with-websockets-and-node-js/ "Real time monitoring PHP applications with websockets and node.js")). If we use raw WebSocket implementations and our connection with the web-socket server crashes or if we stop the server, our application will raise disconnect event and we need to create something to reconnect to the server. socket.io does it for us. With our small piece of JavaScript code we will get a high performance real-time applicatrion. Node is cool. Really cool. Kinda wierd at the beginning but the learning effort will be worthwhile. A few js lines and a real time applications. I’ve got a problem within our node.js application. If we’ve got some kind of security within our application (imagine for example it’s behind a session based auth form) we need to share this security layer with our node.js server to ensure that non authenticated users aren’t allowed to use our websockets. I don’t know how to do it just now, but I’m investigating. Do you have any idea?

Full code Code at [github](https://github.com/gonzalo123/socket.io.test). Ensure you're using the stable version of node.js. With the last versión available on github of node.js there's a bug and server dies when we connect with Google Chrome.
