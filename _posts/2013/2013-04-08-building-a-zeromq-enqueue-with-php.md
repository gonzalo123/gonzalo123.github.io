---
layout: post
title: "Enqueue Symfony's process components with PHP and ZeroMQ"
date: "2013-04-08"
tags:
  - "php"
  - "technology"
  - "process"
  - "reactor-pattern"
  - "sysfony"
  - "zeromq"
---

Today I'd like to play with [ZeroMQ](http://www.zeromq.org/). ZeroMQ is a great tool to work with sockets. I will show you the problem that I want to solve: One web application needs to execute background processes but I need to execute those processes in order. Two users cannot execute one process at the same time. OK, if we face to this problem we can use Gearman. I've written various posts about Gearman ([here](http://gonzalo123.com/2011/03/07/watermarks-in-our-images-with-php-and-gearman/ "Watermarks in our images with PHP and Gearman") and [here](http://gonzalo123.com/2010/11/01/database-connection-pooling-with-php-and-gearman/ "Database connection pooling with PHP and gearman") for example). But today I want to play with ZeroMQ.

I'm going to use one great library called [React](http://reactphp.org/). With react (reactor pattern implementation in PHP) we can do various thing. One of them are [ZeroMQ](https://github.com/reactphp/zmq) bindings.

In this simple example we are going to build a simple server and client. The client will send to the server one string that the server will enqueue and executes using the Symfony's Process component.

Here is the client: 

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
 
use Zmqlifo\Client;
 
$queue = Client::factory('tcp://127.0.0.1:4444');
echo $queue->run("ls -latr")->getOutput();
echo $queue->run("pwd")->getOutput();
```

And finally the server: 

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
 
use Symfony\Component\Process\Process;
use Zmqlifo\Server;
 
$server = Server::factory('tcp://127.0.0.1:4444');
$server->registerOnMessageCallback(function ($msg) {
    $process = new Process($msg);
    $process->setTimeout(3600);
    $process->run();
    return $process->getOutput();
});
 
$server->run();
```

You can see the working example here:

[![youtube](https://img.youtube.com/vi/Vmyr5H4eIHc/0.jpg)](https://www.youtube.com/watch?v=Vmyr5H4eIHc)

you can check the full code of the library in [github](https://github.com/gonzalo123/zmqlifo) and [Packagist](https://packagist.org/packages/gonzalo123/zmqlifo).

**UPDATE** As Igor Wiedler [said](https://github.com/gonzalo123/zmqlifo/pull/1) React is not necessary here.

> ZMQ is used for blocking sends and blocking tasks, having an event loop does not really make much sense.

github repository updated (thanks!).
