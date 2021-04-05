---
layout: post
title: "Building a simple TCP proxy server with node.js"
date: "2012-09-17"
tags: 
  - "node-js"
  - "technology"
  - "node"
  - "nodejs"
  - "tcp"
---

Today we are going to build a simple TCP proxy server. The scenario is the following one. We have got one host (the client) that establishes a TCP connection to another one (the remote).

> client ---> remote

We want to set up a proxy server in the middle, so the client will establish the connection with the proxy and the proxy will forward it to the remote, keeping in mind the remote response also. With node.js is really simple to perform those kind of network operations.

> client ---> proxy -> remote

```javascript
var net = require('net');
 
var LOCAL_PORT  = 6512;
var REMOTE_PORT = 6512;
var REMOTE_ADDR = "192.168.1.25";
 
var server = net.createServer(function (socket) {
    socket.on('data', function (msg) {
        console.log('  ** START **');
        console.log('<< From client to proxy ', msg.toString());
        var serviceSocket = new net.Socket();
        serviceSocket.connect(parseInt(REMOTE_PORT), REMOTE_ADDR, function () {
            console.log('>> From proxy to remote', msg.toString());
            serviceSocket.write(msg);
        });
        serviceSocket.on("data", function (data) {
            console.log('<< From remote to proxy', data.toString());
            socket.write(data);
            console.log('>> From proxy to client', data.toString());
        });
    });
});
 
server.listen(LOCAL_PORT);
console.log("TCP server accepting connection on port: " + LOCAL_PORT);
```

Simple, isn't it? Source code in [github](https://github.com/gonzalo123/nodejs.tcp.proxy)
