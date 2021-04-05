---
layout: post
title: "Scroll desktop's Web pages remotely with our smartphone, using Node.js and WebSockets"
date: "2012-07-16"
tags: 
  - "node-js"
  - "socket-io"
  - "technology"
  - "software"
  - "technology"
---

Why this script? OK. It was a crazy idea. It started with one “_Is it possible? Yes, let’s code it_” in my mind. Let start. I want to scroll one web page in the TV's web browser (or PC's browser) using my smartphone lying on my couch. I’ve got a wireless mouse so I don’t really need it, but scroll the TV browser with the smartphone sounds cool, isn’t it?

The idea is the following one:

- One QR code in our web page (added dinamically with JavaScrip with Google's Chart API ). Write urls with the smartphone is hard and QR has a good hype, so we will add a QR code at the bottom of the web page with the link to the node.js server.

![](/assets/images/qr1.png)

- One socket.io server built with a node.js server for the real time communications. This node.js server will serve also a jQuery Mobile application with four buttons (with [express](http://expressjs.com/) and [jade](http://jade-lang.com/)):

![](/assets/images/jquerymobile.png "jqueryMobile")

- The server will register the WebSocket and send the real time commands to the browser (with one easy-to-hack security token).
- The browser will handle the socket.io actions and controls the scroll of the web page.

The code is probably crowded by bugs and security problems, but it works and it was enough in my experiment :) :

The node.js server: 

```javascript
var io = require('socket.io').listen(8080);
 
var app = require('express').createServer();
app.set('view engine', 'jade');
 
app.set('view options', {
    layout:false
});
 
app.get('/panel/:key', function (req, res) {
    var key = req.params.key;
    console.log(key);
    res.render('mobile.jade', {key:key});
});
 
app.get('/action/:key/:y/:action', function (req, res) {
    var key = req.params.key;
    var y = req.params.y;
    var action = req.params.action;
    sockets[key].emit('scrollTo', {y:y, action:action});
    res.send('OK');
});
 
app.listen(8000);
 
var sockets = {};
io.sockets.on('connection', function (socket) {
    socket.on('setKey', function (key) {
        sockets[key] = socket;
    });
});
```

The jade template with the jquery mobile application: 

```yaml
!!! 5
html
    head
        meta(charset="utf-8")
        meta(name="viewport", content="width=device-width, initial-scale=1")
        title title
        link(rel='stylesheet', href='http://code.jquery.com/mobile/1.1.0/jquery.mobile-1.1.0.min.css')
        script(src='http://ajax.googleapis.com/ajax/libs/jquery/1.7.1/jquery.min.js')
        script(src='http://code.jquery.com/mobile/1.1.0/jquery.mobile-1.1.0.min.js')
        script(type='text/javascript')
            $(document).bind('pageinit', function() {
                $('#toTop').tap(function() {
                    $.get('/action/#{key}/0/go', function() {});
                });
                $('#toBotton').tap(function() {
                    $.get('/action/#{key}/max/go', function() {});
                });
                $('#toUp').tap(function() {
                    $.get('/action/#{key}/100/rew', function() {});
                });
                $('#toDown').tap(function() {
                    $.get('/action/#{key}/100/ffd', function() {});
                });
            });
    body
        #page1(data-role="page")
            #header(data-theme="a", data-role="header")
                h3 Header
            #content(data-role="content")
                a(data-role="button", data-transition="fade", data-theme="a", href="#", id="toTop", data-icon="minus", data-iconpos="left") Top
                a(data-role="button", data-transition="fade", href="#", id="toUp", data-icon="arrow-u", data-iconpos="left") Up
                a(data-role="button", data-transition="fade", href="#", id="toDown", data-icon="arrow-d", data-iconpos="left") Down
                a(data-r
```

Our Html client page 

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
        "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
    <title>jQuery Smooth Scroll - Design Woop</title>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
</head>
<body>
<p>
    Lorem ipsum ….. put a big lorem ipsum here to make possible the scroll
</p>
<script src="http://localhost:8080/socket.io/socket.io.js"></script>
<script src="client.js"></script>
</body>
</html>
```

And finally our client magic with the QR code and the functions to handle the node.js actions

```javascript
var key = "secret";
function getDocHeight() {
    var D = document;
    return Math.max(
            Math.max(D.body.scrollHeight, D.documentElement.scrollHeight),
            Math.max(D.body.offsetHeight, D.documentElement.offsetHeight),
            Math.max(D.body.clientHeight, D.documentElement.clientHeight)
    );
}
var socket = io.connect('http://localhost:8080');
var y = 0;
 
socket.emit('setKey', key);
socket.on('scrollTo', function (data) {
    if (data.y == 'max') {
        y = getDocHeight();
    } else {
        if (data.action == 'ffd') {
            y += parseInt(data.y);
        } else if (data.action == 'go') {
            y = parseInt(data.y);
        } else {
            y -= parseInt(data.y);
        }
    }
    window.scrollTo(0, y);
 
});
document.write('<img src="https://chart.googleapis.com/chart?chs=150x150&cht=qr&chl=http://192.168.2.3:8000/panel/' + key + '&choe=UTF-8" alt=""/>')
```

You can see the code in [github](https://github.com/gonzalo123/scroll.io) too. We also can see the script in action here:

[![youtube](https://img.youtube.com/vi/tPxjxS198vE\/0.jpg)](https://www.youtube.com/watch?v=tPxjxS198vE\)

We can also add more features to our application but that's enought for this experiment. What do you think?
