---
layout: post
title: "Enclosing socket.io Websocket connection inside a HTML5 SharedWorker"
date: "2014-12-01"
tags: 
  - "html5"
  - "node-js"
  - "socket-io"
  - "technology"
  - "js"
  - "nodejs"
  - "websockets"
---

I really like WebSockets. I've written several posts about [them](http://gonzalo123.com/category/technology/websockets/). Today we're going to speak about something related to WebSockets. Let me explain it a little bit.

Imagine that we build a Web Application with WebSockets. That's means that when we start the application, we need to connect to the WebSockets server. If our application is a Single-page application, we'll create one socket per application, but: What happens if we open three tabs with the application within the browser? The answer is simple, we'll create three sockets. Also, if we reload one tab (a full refresh) we'll disconnect our socket and reconnect again. Maybe we can handle this situation, but we can easily bypass this disconnect-connect situation with a HTML5 feature called [SharedWorkers](http://www.w3.org/TR/workers/#shared-workers-introduction).

Web Workers allows us to run JavaScript process in background. We also can create Shared Workers. SharedWorkers can be shared within our browser session. That's means that we can enclose our WebSocket server inside s SharedWorker, and if we open various tabs with our browser we only need one Socket (one socket per session instead one socket per tab).

I've written a simple library called gio to perform this operation. gio uses socket.io to create WebSockets. WebWorker is a new HTML5 feature and it needs a modern browser. Socket.io works also with old browsers. It checks if WebWorkers are available and if they isn't, then gio creates a WebSocket connection instead of using a WebWorker to enclose the WebSockets.

We can see one simple video to see how it works. In the video we can see how sockets are created. Only one socket is created even if we open more than one tab in our browser. But if we open a new session (one incognito session for example), a new socket is created 

[![youtube](https://img.youtube.com/vi/hirV-ZXHaHc/0.jpg)](https://www.youtube.com/watch?v=hirV-ZXHaHc)

Here we can see the SharedWorker code: 

```javascript
"use strict";
 
importScripts('socket.io.js');
 
var socket = io(self.name),
    ports = [];
 
addEventListener('connect', function (event) {
    var port = event.ports[0];
    ports.push(port);
    port.start();
 
    port.addEventListener("message", function (event) {
        for (var i = 0; i < event.data.events.length; ++i) {
            var eventName = event.data.events[i];
 
            socket.on(event.data.events[i], function (e) {
                port.postMessage({type: eventName, message: e});
            });
        }
    });
});
 
socket.on('connect', function () {
    for (var i = 0; i < ports.length; i++) {
        ports[i].postMessage({type: '_connect'});
    }
});
 
socket.on('disconnect', function () {
    for (var i = 0; i < ports.length; i++) {
        ports[i].postMessage({type: '_disconnect'});
    }
});
```

And here we can see the gio source code:

```javascript
var gio = function (uri, onConnect, onDisConnect) {
    "use strict";
    var worker, onError, workerUri, events = {};
 
    function getKeys(obj) {
        var keys = [];
 
        for (var i in obj) {
            if (obj.hasOwnProperty(i)) {
                keys.push(i);
            }
        }
 
        return keys;
    }
 
    function onMessage(type, message) {
        switch (type) {
            case '_connect':
                if (onConnect) onConnect();
                break;
            case '_disconnect':
                if (onDisConnect) onDisConnect();
                break;
            default:
                if (events[type]) events[type](message);
        }
    }
 
    function startWorker() {
        worker = new SharedWorker(workerUri, uri);
        worker.port.addEventListener("message", function (event) {
            onMessage(event.data.type, event.data.message);
 
        }, false);
 
        worker.onerror = function (evt) {
            if (onError) onError(evt);
        };
 
        worker.port.start();
        worker.port.postMessage({events: getKeys(events)});
    }
 
    function startSocketIo() {
        var socket = io(uri);
        socket.on('connect', function () {
            if (onConnect) onConnect();
        });
 
        socket.on('disconnect', function () {
            if (onDisConnect) onDisConnect();
        });
 
        for (var eventName in events) {
            if (events.hasOwnProperty(eventName)) {
                socket.on(eventName, socketOnEventHandler(eventName));
            }
        }
    }
 
    function socketOnEventHandler(eventName) {
        return function (e) {
            onMessage(eventName, e);
        };
    }
 
    return {
        registerEvent: function (eventName, callback) {
            events[eventName] = callback;
        },
 
        start: function () {
            if (!SharedWorker) {
                startSocketIo();
            } else {
                startWorker();
            }
        },
 
        onError: function (cbk) {
            onError = cbk;
        },
 
        setWorker: function (uri) {
            workerUri = uri;
        }
    };
};
```

And here the application code:

```javascript
(function (gio) {
    "use strict";
 
    var onConnect = function () {
        console.log("connected!");
    };
 
    var onDisConnect = function () {
        console.log("disconnect!");
    };
 
    var ws = gio("http://localhost:8080", onConnect, onDisConnect);
    ws.setWorker("sharedWorker.js");
 
    ws.registerEvent("message", function (data) {
        console.log("message", data);
    });
 
    ws.onError(function (data) {
        console.log("error", data);
    });
 
    ws.start();
}(gio));
```

I've also created a simple webSocket server with socket.io. In this small server there's a setInterval function broadcasting one message to all clients per second to see the application working

```javascript
var io, connectedSockets;
 
io = require('socket.io').listen(8080);
connectedSockets = 0;
 
io.sockets.on('connection', function (socket) {
    connectedSockets++;
    console.log("Socket connected! Conected sockets:", connectedSockets);
 
    socket.on('disconnect', function () {
        connectedSockets--;
        console.log("Socket disconnect! Conected sockets:", connectedSockets);
    });
});
 
setInterval(function() {
    io.emit("message", "Hola " + new Date().getTime());
}, 1000); 
```

Source code is available in my [github account](https://github.com/gonzalo123/wsww).
