---
layout: post
title: "Silex service provider for a Gearman wrapper"
date: "2016-10-19"
tags: 
  - "php"
  - "technology"
  - "gearman"
  - "queues"
  - "serviceprovider"
  - "silex"
---

I've written an small wrapper for the [gearman](http://gearman.org/) api. Normally I use [Silex](http://silex.sensiolabs.org/) in the frontend. Today we're going to build a service provider to allow us to integrate the gearman wrapper easily within our Silex applications.

Here I show you an example of how to use the service provider.

Imagine this simple worker: 

```php
use G\Gearman\Builder;
 
$worker = Builder::createWorker();
 
$worker->on("worker.example", function ($response) {
    return strrev($response);
});
 
$worker->run();
```

And this is the Silex application using the service provider as a gearman client:

```php
use G\Gearman\Client;
use G\GearmanServiceProvider;
use Silex\Application;
 
$app = new Application();
 
$app->register(new GearmanServiceProvider());
 
$app->get("/", function (Client $client) {
    return "Hello " . $client->doNormal("worker.example", "Gonzalo");
});
 
$app->run();

```

I'm using [injector](https://github.com/gonzalo123/injector) library to inject providers. I've written about it [here](https://gonzalo123.com/2015/10/19/alternative-way-to-inject-providers-in-a-silex-application/).

This is the code of the service provider 

```php
namespace G;
use G\Gearman\Client;
use G\Gearman\Tasks;
use Injector\InjectorServiceProvider;
use Silex\Application;
use Silex\ServiceProviderInterface;
class GearmanServiceProvider implements ServiceProviderInterface
{
    private $client;
    public function __construct(\GearmanClient $client = null)
    {
        if (is_null($client)) {
            $client = new \GearmanClient();
            $client->addServers("localhost:4730");
        }
        $this->client = $client;
    }
    public function register(Application $app)
    {
        $app->register(new InjectorServiceProvider([
            'G\Gearman\Client' => 'gearmanClient',
            'G\Gearman\Tasks'  => 'gearmanTasks',
        ]));
        $app['gearmanClient'] = function () use ($app) {
            $client = new Client($this->client);
            $client->onSuccess(function ($response) {
                return $response;
            });
            return $client;
        };
        $app['gearmanTasks'] = function () use ($app) {
            return new Tasks($this->client);
        };
    }
    public function boot(Application $app)
    {
    }
}
```

The code is available in my [github](https://github.com/gonzalo123/gearmanserviceprovider/blob/master/src/GearmanServiceProvider.php) account
