---
layout: post
title: "Managing Windows services with Symfony/Process and PHP"
date: "2012-10-29"
tags: 
  - "php"
  - "technology"
  - "behat"
  - "command-line-parser"
  - "service-status"
  - "services"
  - "software"
  - "symfony"
  - "windows"
---

Sometimes I need to stop/start remote Windows services with PHP. It's quite easy to do it with **_net_** commnand. This command is a tool for administration of Samba and remote CIFS servers. It's pretty straightforward to handle them from Linux command line:

```commandline
net rpc service --help
Usage:
net rpc service list
    View configured Win32 services
net rpc service start
    Start a service
net rpc service stop
    Stop a service
net rpc service pause
    Pause a service
net rpc service resume
    Resume a service
net rpc service status
    View current status of a service
net rpc service delete
    Deletes a service
net rpc service create
    Creates a service
```

Today we are going to create a PHP wrapper for this tool. Our NetService library will have two classes: One Parser and one Service class.

The Parser's responsibility will be create the command line instruction. I will use [Behat](http://behat.org/) in the developing process of Parser class. Here we can see the feature file:

```yaml
Feature: command line parser
 
  Scenario: net service list
    Given windows server host called "windowshost.com"
    And credentials are "myDomanin/user%password"
    And action is "list"
    Then command line is "net rpc service list -S windowshost.com -U myDomanin/user%password"
 
  Scenario: net service start
    Given windows server host called "windowshost.com"
    And service name called "ServiceName"
    And credentials are "myDomanin/user%password"
    And action is "start"
    Then command line is "net rpc service start ServiceName -S windowshost.com -U myDomanin/user%password"
 
  Scenario: net service stop
    Given windows server host called "windowshost.com"
    And service name called "ServiceName"
    And credentials are "myDomanin/user%password"
    And action is "stop"
    Then command line is "net rpc service stop ServiceName -S windowshost.com -U myDomanin/user%password"
 
  Scenario: net service pause
    Given windows server host called "windowshost.com"
    And service name called "ServiceName"
    And credentials are "myDomanin/user%password"
    And action is "pause"
    Then command line is "net rpc service pause ServiceName -S windowshost.com -U myDomanin/user%password"
 
  Scenario: net service resume
    Given windows server host called "windowshost.com"
    And service name called "ServiceName"
    And credentials are "myDomanin/user%password"
    And action is "resume"
    Then command line is "net rpc service resume ServiceName -S windowshost.com -U myDomanin/user%password"
 
  Scenario: net service status
    Given windows server host called "windowshost.com"
    And service name called "ServiceName"
    And credentials are "myDomanin/user%password"
    And action is "status"
    Then command line is "net rpc service status ServiceName -S windowshost.com -U myDomanin/user%password"
```

The implementation of the feature file:

```php
namespace NetService;
 
class Parser
{
    private $host;
    private $credentials;
 
    public function __construct($host, $credentials)
    {
        $this->host        = $host;
        $this->credentials = $credentials;
    }
 
    public function getCommandLineForAction($action, $service = NULL)
    {
        if (!is_null($service)) $service = " {$service}";
        return "net rpc service {$action}{$service} -S {$this->host} -U {$this->credentials}";
    }
}
```

and finally our Service class:

```php
namespace NetService;
 
use Symfony\Component\Process\Process,
    NetService\Parser;
 
class Service
{
    private $parser;
    private $timeout;
 
    const START = 'start';
    const STOP = 'stop';
    const STATUS = 'status';
    const LIST_SERVICES = 'list';
    const PAUSE = 'pause';
    const RESUME = 'resume';
 
    const DEFAULT_TIMEOUT = 3600;
 
    public function __construct(Parser $parser)
    {
        $this->parser = $parser;
        $this->timeout = self::DEFAULT_TIMEOUT;
    }
 
    public function start($service)
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::START, $service));
    }
 
    public function stop($service)
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::STOP, $service));
    }
 
    public function pause($service)
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::PAUSE, $service));
    }
 
    public function resume($service)
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::RESUME, $service));
    }
 
    public function status($service)
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::STATUS, $service));
    }
 
    public function listServices()
    {
        return $this->runProcess($this->parser->getCommandLineForAction(self::LIST_SERVICES));
    }
 
    public function isRunning($service)
    {
        $status = explode("\n", $this->status($service));
        if (isset($status[0]) && strpos(strtolower($status[0]), "running") !== FALSE) {
            return TRUE;
        } else {
            return FALSE;
        }
    }
 
    public function setTimeout($timeout)
    {
        $this->timeout = $timeout;
    }
 
    private function runProcess($commandLine)
    {
        $process = new Process($commandLine);
        $process->setTimeout($this->timeout);
        $process->run();
 
        if (!$process->isSuccessful()) {
            throw new RuntimeException($process->getErrorOutput());
        }
 
        return $process->getOutput();
    }
 
    private function parseStatus($status)
    {
        return explode("\n", $status);
    }
}
```

And that's all. Now a couple of examples:

```php
include __DIR__ . '/../vendor/autoload.php';
 
use NetService\Service,
    NetService\Parser;
 
$host        = 'windowshost.com';
$serviceName = 'ServiceName';
$credentials = '{domain}/{user}%{password}';
 
$service = new Service(new Parser($host, $credentials));
 
if ($service->isRunning($serviceName)) {
    echo "Service is running. Let's stop";
    $service->stop($serviceName);
 
} else {
    echo "Service isn't running. Let's start";
    $service->start($serviceName);
}
 
//dumps status output
echo $service->status($serviceName);
```

```php
include __DIR__ . '/../vendor/autoload.php';
 
use NetService\Service,
    NetService\Parser;
 
$host        = 'windowshost.com';
$credentials = '{domain}/{user}%{password}';
 
$service = new Service(new Parser($host, $credentials));
echo $service->listServices();
```

You can see the full code in [github](https://github.com/gonzalo123/netservice) here. The package is also available for composer at [Packaist](https://packagist.org/packages/gonzalo123/netservice).
