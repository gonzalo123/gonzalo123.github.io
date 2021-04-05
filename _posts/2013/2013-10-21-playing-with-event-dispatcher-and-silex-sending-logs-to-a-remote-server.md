---
layout: post
title: "Playing with event dispatcher and Silex. Sending logs to a remote server."
date: "2013-10-21"
tags: 
  - "php"
  - "symfony"
  - "technology"
  - "silex"
---

Today I [continue](http://gonzalo123.com/2013/10/14/using-the-event-dispatcher-in-a-silex-application/ "Using the event dispatcher in a Silex application") playing with event dispatcher and Silex. Now I want to send a detailed log of our Kernel events to a remote server. We can do it something similar with [Monolog](https://github.com/Seldaek/monolog), but I want to implement one working example hacking a little bit the event dispatcher. Basically we're going to create one Logger class (implementing [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md) of course)

```php
namespace G;
 
use Psr\Log\LoggerInterface;
use Psr\Log\LogLevel;
 
class Logger implements LoggerInterface
{
    private $socket;
 
    public function __construct($socket)
    {
        $this->socket = $socket;
    }
 
    function __destruct()
    {
        @fclose($this->socket);
    }
 
    public function emergency($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::EMERGENCY);
    }
 
    public function alert($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::ALERT);
    }
 
    public function critical($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::CRITICAL);
    }
 
    public function error($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::ERROR);
    }
 
    public function warning($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::WARNING);
    }
 
    public function notice($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::NOTICE);
    }
 
    public function info($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::INFO);
    }
 
    public function debug($message, array $context = array())
    {
        $this->sendLog($message, $context, LogLevel::DEBUG);
    }
 
    public function log($level, $message, array $context = array())
    {
        $this->sendLog($message, $context, $level);
    }
 
    private function sendLog($message, array $context = array(), $level = LogLevel::INFO)
    {
        $data = serialize([$message, $context, $level]);
        @fwrite($this->socket, "{$data}\n");
    }
}
```

As you can see our Logger class send logs to a remote server, with a socket passed within the constructor. We also need one Service Provider called LoggerServiceProvider to integrate our Logger instance into our Silex application.

```php
namespace G;
 
use Silex\Application;
use Silex\ServiceProviderInterface;
 
class LoggerServiceProvider implements ServiceProviderInterface
{
    private $socket;
 
    public function __construct($socket)
    {
        $this->socket = $socket;
    }
 
    public function register(Application $app)
    {
        $app['remoteLogger'] = $app->share(
            function () use ($app) {
                return new Logger($this->socket);
            }
        );
    }
 
    public function boot(Application $app)
    {
    }
}
```

And now the last part is our Silex application:

```php
use G\LoggerServiceProvider;
use G\Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel;
use Symfony\Component\HttpKernel\Event;
 
$app = new Application();
$app->register(new LoggerServiceProvider(stream_socket_client('tcp://localhost:4000')));
 
$app->on(HttpKernel\KernelEvents::REQUEST, function (Event\GetResponseEvent $event) use ($app) {
        $app->getLogger()->info($event->getName());
    }
);
 
$app->on(HttpKernel\KernelEvents::CONTROLLER, function (Event\FilterControllerEvent $event) use ($app) {
        $app->getLogger()->info($event->getName());
    }
);
 
$app->on(HttpKernel\KernelEvents::TERMINATE, function (Event\PostResponseEvent $event) use ($app) {
        $app->getLogger()->info($event->getName());
    }
);
 
$app->on(HttpKernel\KernelEvents::EXCEPTION, function (Event\GetResponseForExceptionEvent $event) use ($app) {
        $app->getLogger()->critical($event->getException()->getMessage());
    }
);
 
$app->get('/', function () {
    return 'Hello';
});
 
$app->run();
```

As we can see the event dispacher send each event to a remote server (in this example: tcp://localhost:4000). Now we only need a tcp server to handle those sockets. We can use different tools and libraries to do that. In this example we're going to use [React](http://reactphp.org/).

```php
use React\EventLoop\Factory;
use React\Socket\Server;
 
$loop   = Factory::create();
$socket = new Server($loop);
 
$socket->on('connection', function (\React\Socket\Connection $conn){
    $unique = uniqid();
    $conn->on('data', function ($data) use ($unique) {
            list($message, $context, $level) = \unserialize($data);
            echo date("d/m/Y H:i:s")."::{$level}::{$unique}::{$message}" . PHP_EOL;
        });
});
 
echo "Socket server listening on port 4000." .PHP_EOL;
echo "You can connect to it by running: telnet localhost 4000" . PHP_EOL;
 
$socket->listen(4000);
$loop->run();
```

Now we only need to start our servers: our silex one

```commandline
php -S 0.0.0.0:8080 -t www
```

and the tcp server 

```commandline
php app/server.php
```

One screencast showing the prototype in action:
[![youtube](https://img.youtube.com/vi/4Dr3Yz4OcpI/0.jpg)](https://www.youtube.com/watch?v=4Dr3Yz4OcpI)

You can see the full code in my [github](https://github.com/gonzalo123/silexlisteners) account.
