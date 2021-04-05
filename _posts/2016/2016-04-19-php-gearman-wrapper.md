---
layout: post
title: "PHP Gearman Wrapper"
date: "2016-04-19"
categories:
tags:
  - "php"
  - "technology"
  - "gearman"
  - "queues"
---

I must admit that nowadays I'm using RabbitMQ more than Gearman but I'm still a big fan of [gearman](https://gonzalo123.com/tag/gearman/). PHP has a great api to connect to [gearman](http://gearman.org/) work server but sometimes I miss another, how to say, "clean" way. Because of that I've creates a gearman wrapper. Let's start.

I want to cover different [areas](http://gearman.org/manual/): Workers, clients, background clients, and tasks.

Worker example: 

```php
use G\Gearman\Builder;
 
$worker = Builder::createWorker();
 
$worker->on("slow.process", function ($response, \GearmanJob $job) {
    echo "Response: {$response} unique: {$job->unique()}\n";
    // we emulate a slow process with a sleep
    sleep(2);
 
    return $job->unique();
});
 
$worker->on("fast.process", function ($response, \GearmanJob $job) {
    echo "Response: {$response} unique: {$job->unique()}\n";
 
    return $job->unique();
});
 
$worker->on("exception.process", function () {
    // we emulate a failing process
    throw new \Exception("Something wrong happens");
});
 
$worker->run();
```

And a client: 

```php
use G\Gearman\Builder;
 
$client = Builder::createClient();
 
$client->onSuccess(function ($response) {
    echo $response;
});
 
$client->doNormal('fast.process', "Hello");
```

One background client 

```php
use G\Gearman\Builder;
 
$client = Builder::createClient();
 
$client->doBackground('slow.process', "Hello1");
$client->doBackground('slow.process', "Hello2");
$client->doBackground('slow.process', "Hello3");
```

And finally, tasks 

```php
use G\Gearman\Builder;
 
$tasks = Builder::createTasks();
 
$tasks->onSuccess(function (\GearmanTask $task, $context) {
    $out = is_callable($context) ? $context($task) : $task->data();
    echo "onSuccess response: " . $out . " id: {$task->unique()}\n";
});
 
$tasks->onException(function (\GearmanTask $task) {
    echo "onException response {$task->data()}\n";
});
 
$responseParser = function (\GearmanTask $task) {
    return "Hello " . $task->data();
};
 
$tasks->addTask('fast.process', "fast1", $responseParser, 'g1');
$tasks->addTaskHigh('slow.process', "slow1", null, 'xxxx');
$tasks->addTask('fast.process', "fast2");
$tasks->addTask('exception.process', 'hi');
 
$tasks->runTasks();
```

The library is just a wrapper to the official api. I've create a builder to simplify the creation of the instances:

```php
namespace G\Gearman;
class Builder
{
    static function createWorker($servers=null)
    {
        $worker = new \GearmanWorker();
        $worker->addServers($servers);
        return new Worker($worker);
    }
    static function createClient($servers=null)
    {
        $client = new \GearmanClient();
        $client->addServers($servers);
        return new Client($client);
    }
    static function createTasks($servers=null)
    {
        $client = new \GearmanClient();
        $client->addServers($servers);
        return new Tasks($client);
    }
}
```

that's the worker wrapper 

```php
namespace G\Gearman;
class Worker
{
    private $worker;
    public function __construct(\GearmanWorker $worker)
    {
        $this->worker = $worker;
    }
    public function on($name, callable $callback, $context = null, $timeout = 0)
    {
        $this->worker->addFunction($name, function (\GearmanJob $job) use ($callback) {
            return call_user_func($callback, json_decode($job->workload()), $job);
        }, $context, $timeout);
    }
    public function run()
    {
        try {
            $this->loop();
        } catch (\Exception $e) {
            echo $e->getMessage() . "\n";
            $this->run();
        }
    }
    private function loop()
    {
        while ($this->worker->work()) {
        }
    }
}
```

Now the client one 

```php
namespace G\Gearman;
class Client
{
    private $onSuccessCallback;
    private $client;
    public function __construct(\GearmanClient $client)
    {
        $this->client = $client;
    }
    public function doHigh($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    public function doNormal($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    public function doLow($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    public function doBackground($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    public function doHighBackground($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    public function doLowBackground($name, $workload=null, $unique = null)
    {
        return $this->doAction(__FUNCTION__, $name, $workload, $unique);
    }
    private function doAction($action, $name, $workload=null, $unique)
    {
        $workload = (string)$workload;
        $handle = $this->client->$action($name, json_encode($workload), $unique);
        $returnCode = $this->client->returnCode();
        if ($returnCode != \GEARMAN_SUCCESS) {
            throw new \Exception($this->client->error(), $returnCode);
        } else {
            if ($this->onSuccessCallback) {
                return call_user_func($this->onSuccessCallback, $handle);
            }
        }
        return null;
    }
    public function onSuccess(callable $callback)
    {
        $this->onSuccessCallback = $callback;
    }
}
```

and finally the tasks 

```php
namespace G\Gearman;
class Tasks
{
    private $client;
    private $tasks;
    public function __construct(\GearmanClient $client)
    {
        $this->tasks = [];
        $this->client = $client;
    }
    public function addTask($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function addTaskHigh($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function addTaskLow($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function addTaskBackground($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function addTaskHighBackground($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function addTaskLowBackground($name, $workload=null, $context = null, $unique = null)
    {
        $this->tasks[] = [__FUNCTION__, $name, $workload, $context, $unique];
    }
    public function runTasks()
    {
        foreach ($this->tasks as list($actionName, $name, $workload, $context, $unique)) {
            $this->client->$actionName($name, json_encode($workload), $context, $unique);
        }
        $this->client->runTasks();
    }
    public function onSuccess(callable $callback)
    {
        $this->client->setCompleteCallback($callback);
    }
    public function onException(callable $callback)
    {
        $this->client->setExceptionCallback($callback);
    }
    public function onFail(callable $callback)
    {
        $this->client->setFailCallback($callback);
    }
}
```

Library is available in [packagist](https://packagist.org/packages/gonzalo123/gearman) and source code in my [github](https://github.com/gonzalo123/gearman) account.
