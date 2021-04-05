---
title: "Building TCP server daemond with PHP and Ratchet"
date: "2015-04-13"
tags: 
  - "php"
  - "silex"
  - "technology"
  - "ratchet"
  - "socket"
  - "tcp"
  - "xinet-d"
---

In my daily work I normally play a lot with TCP servers, clients and things like that. I like to use Linux's [xinet.d](http://en.wikipedia.org/wiki/Xinetd) daemon to handle the TCP ports.

I've also written [something](http://gonzalo123.com/2010/05/23/building-network-services-with-php-and-xinetd/ "Building network services with PHP and xinetd") about it. This approach works fine. You don't need to open any port. Xinet.d opens the ports and invoke the PHP scripts. The problem appears when we call intensively our xinet.d server. It creates one PHP instance per request. It isn't a problem with one request in, for example, 3 seconds, but if we need to handle 10 requests per second our server load will grow. The solution: a dedicated server.

With PHP we can create dedicated servers using, for example, [Ratchet](http://socketo.me/). I want to create a library using Ratchet to open TCP ports and register callbacks to those ports to handle the requests (Reactor pattern). Do you know [Silex](http://silex.sensiolabs.org/)? Of course you know. This library borrows the idea of Silex (register callbacks to routes) to the TCP world.

Let me show examples:

**Example 1:** 
```php
use React\EventLoop\Factory as LoopFactory;
use G\Pxi\Pxinetd;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$service->on(8080, function ($data) {
    echo $data;
});
 
$service->register($loop);
$loop->run();
```

That's the simplest example. A TCP echo server. We open 8080 port to all interfaces (0.0.0.0) and we return a simple input echo.

We can start different ports also:

```php
use G\Pxi\Pxinetd;
use React\EventLoop\Factory as LoopFactory;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$service->on(8080, function ($data) {
    echo $data;
});
 
$service->on(8888, function ($data) {
    echo $data;
});
 
$service->register($loop);
$loop->run();
```

**Example 2:** 
We can also work with the connection

```php
use G\Pxi\Pxinetd;
use G\Pxi\Connection;
use React\EventLoop\Factory as LoopFactory;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$service->on(8080, function ($data) {
    echo $data;
});
 
$service->on(8088, function ($data, Connection $conn) {
    var_dump($conn->getRemoteAddress());
    echo $data;
    $conn->send("....");
    $conn->close();
});
 
$service->register($loop);
$loop->run();
```

**Example 3:** 
I'm a big fan of YAML configurations, so we can load configurations from a YAML file, of course:

conf3.yml: 
```yaml
ports:
  9999:
    class: Services\Reader1
```

```php
// Services/Reader1.php
use G\Pxi\Connection;
use G\Pxi\MessageIface;
 
class Reader1 implements MessageIface
{
    public function onMessage($data, Connection $conn)
    {
        echo $data . $conn->getRemoteAddress();
    }
}
```

```php
use G\Pxi\Pxinetd;
use G\Pxi\YamlFileLoader;
use Symfony\Component\Config\FileLocator;
use React\EventLoop\Factory as LoopFactory;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$loader = new YamlFileLoader($service, new FileLocator(__DIR__ ));
$loader->load('conf3.yml');
 
$service->on(8080, function ($data) {
    echo "$data";
});
 
$service->register($loop);
$loop->run();
```

**Example 4:** 
We're using symfony/config and symfony/yaml components, so we can use hierarchy within our yaml files:

```php
use G\Pxi\Pxinetd;
use G\Pxi\YamlFileLoader;
use Symfony\Component\Config\FileLocator;
use React\EventLoop\Factory as LoopFactory;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$loader = new YamlFileLoader($service, new FileLocator(__DIR__));
$loader->load('conf4.yml');
 
$service->on(8080, function ($data) {
    echo "$data";
});
 
$service->register($loop);
$loop->run();
```

config4.yml: 

```yaml
imports:
  - { resource: conf4_2.yml }
ports:
  9999:
    class: Services\Reader1
```

config4\_2.yml 

```yaml
ports:
  7777:
    class: Services\Reader1
```

**Example 5:** 
And finally one bonus. This script is single thread. That means if one process takes too much time it will block to the rest of the processes. We can implemente threads, but I try to avoid them like the plague. I prefer to create a Silex app (behind a http server) and perform http requests to "emulate" threads in a simply way.

```php
use G\Pxi\Pxinetd;
use G\Pxi\YamlFileLoader;
use Symfony\Component\Config\FileLocator;
use React\EventLoop\Factory as LoopFactory;
 
$loop = LoopFactory::create();
$service = new Pxinetd('0.0.0.0');
 
$loader = new YamlFileLoader($service, new FileLocator(__DIR__ ));
$loader->load('conf5.yml');
 
$service->on(8080, function ($data) {
    echo "$data";
});
 
$service->register($loop);
$loop->run();
```

conf5.yml 
```yaml
ports:
  9999:
    class: Services\Reader1
  9991:
    url: http://localhost:8899/onMessage/{data}
  9992:
    url: http://localhost:8899/simulateError/{data}
```

And now our Silex server running at 8899 port: 

```php
include __DIR__ . "/../../vendor/autoload.php";
 
use Silex\Application;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
 
$app = new Application();
 
$app->get('/onMessage/{data}', function ($data) {
    return "OK" . "'{$data}'";
});
 
$app->get('/simulateError/{data}', function ($data) {
    throw new NotFoundHttpException();
});
 
$app->run();
```

And that's all. What do you think? You can see the whole library in my [github](https://github.com/gonzalo123/pxinetd) account.
