---
layout: post
title: "How to send the output of Symfony’s process Component to a node.js server in Real Time with Socket.io"
date: "2012-10-08"
tags: 
  - "node-js"
  - "php"
  - "socket-io"
  - "technology"
---

Today another crazy idea. Do you know Symfony Process Component? The [Process](http://symfony.com/doc/2.0/components/process.html) Component is a simple component that executes commands in sub-processes. I like to use it when I need to execute commands in the operating system. The documentation is pretty straightforward. Normally when I want to collect the output of the script (imagine we run those scripts within a crontab) I save the output in a log file and I can check it even in real time with _tail -f_ command.

This approach works, but I want to do it in a browser (call me crazy :)). I’ve written a [couple](http://gonzalo123.com/2011/05/09/real-time-monitoring-php-applications-with-websockets-and-node-js/) of [posts](http://gonzalo123.com/2011/05/16/web-console-with-node-js/) with something similar. What I want to do now? The idea is simple:

First I want to execute custom commands with process. Just follow the process documentation:

```php
<?php
use Symfony\Component\Process\Process;
 
$process = new Process('ls -lsa');
$process->setTimeout(3600);
$process->run();
if (!$process->isSuccessful()) {
    throw new \RuntimeException($process->getErrorOutput());
}
 
print $process->getOutput();
```

Process has one cool feature, we can give feedback in real-time by passing an anonymous function to the run() method:

```php
<?php
use Symfony\Component\Process\Process;
 
$process = new Process('ls -lsa');
$process->run(function ($type, $buffer) {
    if ('err' === $type) {
        echo 'ERR > '.$buffer;
    } else {
        echo 'OUT > '.$buffer;
    }
});
```

The idea now is to use this callback to send a TCP socket to one server with node.js

```javascript
var LOCAL_PORT = 5600;
var server = require('net').createServer(function (socket) {
    socket.on('data', function (msg) {
        console.log(msg);
    });
}).listen(LOCAL_PORT);
 
server.on('listening', function () {
    console.log("TCP server accepting connection on port: " + LOCAL_PORT);
});
```

Now we change our php script to

```php
<?php
include __DIR__ . "/../vendor/autoload.php";
 
use Symfony\Component\Process\Process;
 
function runProcess($command)
{
    $address = 'localhost';
    $port = 5600;
    $socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
    socket_connect($socket, $address, $port);
 
    $process = new Process($command);
    $process->setTimeout(3600);
    $process->run(
        function ($type, $buffer) use ($socket) {
            if ('err' === $type) {
                socket_write($socket, "ERROR\n", strlen("ERROR\n"));
                socket_write($socket, $buffer, strlen($buffer));
            } else {
 
                socket_write($socket, $buffer, strlen($buffer));
            }
        }
    );
    if (!$process->isSuccessful()) {
        throw new \RuntimeException($process->getErrorOutput());
    }
    socket_close($socket);
}
 
runProcess('ls -latr /');
```

Now with the node.js started, if we run the php script, we will see the output of the process command in the node’s terminal. But we want to show it in a browser. What can we do? Of course, [socket.io](http://socket.io/). We change the node.js command to:

```javascript
var LOCAL_PORT = 5600;
var SOKET_IO_PORT = 8000;
var ioClients = [];
 
var io = require('socket.io').listen(SOKET_IO_PORT);
 
var server = require('net').createServer(function (socket) {
    socket.on('data', function (msg) {
        ioClients.forEach(function (ioClient) {
            ioClient.emit('log', msg.toString().trim());
        });
    });
}).listen(LOCAL_PORT);
 
io.sockets.on('connection', function (socketIo) {
    ioClients.push(socketIo);
});
 
server.on('listening', function () {
    console.log("TCP server accepting connection on port: " + LOCAL_PORT);
});
```

and finally we create a simple web client:

```html
<script src="http://localhost:8000/socket.io/socket.io.js"></script>
<script>
    var socket = io.connect('http://localhost:8000');
    socket.on('log', function (data) {
        console.log(data);
    });
</script>
```

Now if we start one browser we will see the output of our command line process within the console tab of the browser.

![](/assets/images/webconsole23.png)

If you want something like [webConsole](https://github.com/gonzalo123/webConsole), I also have created the example, With a Web UI enabling to send custom commands. You can see it in [github](https://github.com/gonzalo123/webConsole2).

Obviously that’s only one experiment with a lot of security issues that we need to take into account if we want to use it in production. What do you think?
