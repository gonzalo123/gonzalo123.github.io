---
layout: post
title: "Sending logs to a remote server using RabbitMQ"
date: "2016-03-14"
categories: 
  - "php"
  - "technology"
tags: 
  - "queue"
  - "rabbitmq"
  - "silex"
---

Time ago I wrote an [article](http://gonzalo123.com/2013/10/21/playing-with-event-dispatcher-and-silex-sending-logs-to-a-remote-server/) to show how to send Silex logs to a remote server. Today I want to use a messaging queue to do it. Normally, when I need queues, I use [Gearman](http://gearman.org/) but today I want to play with [RabbitMQ](https://www.rabbitmq.com/).

When we work with web applications it's important to have, in some way or another, one way to decouple operations from the main request. Messaging queues are great tools to perform those operations. They even allow us to create our workers with a different languages than the main request. This days, for example, I'm working with [modbus](http://www.modbus.org/) devices. The whole modbus logic is written in Python and I want to use a Frontend with PHP. I can rewrite the modbus logic with PHP (there're PHP libraries to connect with modbus devices), but I'm not so crazy. Queues are our friends.

The idea in this post is the same than the previous post. We'll use event dispatcher to emit events and we'll send those events to a RabitMQ queue. We'll use a Service Provider called.

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
 
use PhpAmqpLib\Connection\AMQPStreamConnection;
use RabbitLogger\LoggerServiceProvider;
use Silex\Application;
use Symfony\Component\HttpKernel\Event;
use Symfony\Component\HttpKernel\KernelEvents;
 
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel    = $connection->channel();
 
$app = new Application(['debug' => true]);
$app->register(new LoggerServiceProvider($connection, $channel));
 
$app->on(KernelEvents::TERMINATE, function (Event\PostResponseEvent $event) use ($app) {
    $app['rabbit.logger']->info('TERMINATE');
});
 
$app->on(KernelEvents::CONTROLLER, function (Event\FilterControllerEvent $event) use ($app) {
    $app['rabbit.logger']->info('CONTROLLER');
});
 
$app->on(KernelEvents::EXCEPTION, function (Event\GetResponseForExceptionEvent $event) use ($app) {
    $app['rabbit.logger']->info('EXCEPTION');
});
 
$app->on(KernelEvents::FINISH_REQUEST, function (Event\FinishRequestEvent $event) use ($app) {
    $app['rabbit.logger']->info('FINISH_REQUEST');
});
 
$app->on(KernelEvents::RESPONSE, function (Event\FilterResponseEvent $event) use ($app) {
    $app['rabbit.logger']->info('RESPONSE');
});
 
$app->on(KernelEvents::REQUEST, function (Event\GetResponseEvent $event) use ($app) {
    $app['rabbit.logger']->info('REQUEST');
});
 
$app->on(KernelEvents::VIEW, function (Event\GetResponseForControllerResultEvent $event) use ($app) {
    $app['rabbit.logger']->info('VIEW');
});
 
$app->get('/', function (Application $app) {
    $app['rabbit.logger']->info('inside route');
    return "HELLO";
});
 
$app->run();
```

Here we can see the service provider:

```php
<?php
namespace RabbitLogger;
 
use PhpAmqpLib\Channel\AMQPChannel;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use Silex\Application;
use Silex\ServiceProviderInterface;
 
class LoggerServiceProvider implements ServiceProviderInterface
{
    private $connection;
    private $channel;
 
    public function __construct(AMQPStreamConnection $connection, AMQPChannel $channel)
    {
        $this->connection = $connection;
        $this->channel    = $channel;
    }
 
    public function register(Application $app)
    {
        $app['rabbit.logger'] = $app->share(
            function () use ($app) {
                $channelName = isset($app['logger.channel.name']) ? $app['logger.channel.name'] : 'logger.channel';
                return new Logger($this->connection, $this->channel, $channelName);
            }
        );
    }
 
    public function boot(Application $app)
    {
    }
}
```

And here the logger: 

```php
<?php
namespace RabbitLogger;
 
use PhpAmqpLib\Channel\AMQPChannel;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
use Psr\Log\LoggerInterface;
use Psr\Log\LogLevel;
use Silex\Application;
 
class Logger implements LoggerInterface
{
    private $connection;
    private $channel;
    private $queueName;
 
    public function __construct(AMQPStreamConnection $connection, AMQPChannel $channel, $queueName = 'logger')
    {
        $this->connection = $connection;
        $this->channel    = $channel;
        $this->queueName  = $queueName;
        $this->channel->queue_declare($queueName, false, false, false, false);
    }
 
    function __destruct()
    {
        $this->channel->close();
        $this->connection->close();
    }
 
    public function emergency($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::EMERGENCY);
    }
 
    public function alert($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::ALERT);
    }
 
    public function critical($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::CRITICAL);
    }
 
    public function error($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::ERROR);
    }
 
    public function warning($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::WARNING);
    }
 
    public function notice($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::NOTICE);
    }
 
    public function info($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::INFO);
    }
 
    public function debug($message, array $context = [])
    {
        $this->sendLog($message, $context, LogLevel::DEBUG);
    }
    public function log($level, $message, array $context = [])
    {
        $this->sendLog($message, $context, $level);
    }
 
    private function sendLog($message, array $context = [], $level = LogLevel::INFO)
    {
        $msg = new AMQPMessage(json_encode([$message, $context, $level]), ['delivery_mode' => 2]);
        $this->channel->basic_publish($msg, '', $this->queueName);
    }
}
```

And finally the RabbitMQ Worker to process our logs

```php
require_once __DIR__ . '/../vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
$channel->queue_declare('logger.channel', false, false, false, false);
echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
$callback = function($msg){
    echo " [x] Received ", $msg->body, "\n";
    //$msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};
//$channel->basic_qos(null, 1, null);
$channel->basic_consume('logger.channel', '', false, false, false, false, $callback);
while(count($channel->callbacks)) {
    $channel->wait();
}
$channel->close();
$connection->close();
```

To run the example we must:

Start RabbitMQ server 

```commandline
rabbitmq-server
```

start Silex server

```commandline
php -S 0.0.0.0:8080 -t www
```

start worker

```commandline
php worker/worker.php
```

You can see whole [project](https://github.com/gonzalo123/rabbitmqLoggerServiceProvider) in my github account
