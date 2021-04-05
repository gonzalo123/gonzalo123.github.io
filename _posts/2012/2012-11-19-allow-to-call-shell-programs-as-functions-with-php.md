---
layout: post
title: "How to call shell programs as functions with PHP"
date: "2012-11-19"
tags: 
  - "php"
  - "symfony2"
  - "technology"
  - "symfony"
---

I'm a big fan of Symfony's [Process](http://symfony.com/doc/2.0/components/process.html) Component. I've used intensively this component within a project and I noticed that I needed a wrapper to avoid to write again and again the same code. Suddenly a cool python library came to my head: [sh](http://amoffat.github.com/sh/). With python's sh we can call any program as if it were a function:

```python
from sh import ifconfig
print(ifconfig("wlan0"))
```

Outputs:

```commandline
wlan0   Link encap:Ethernet  HWaddr 00:00:00:00:00:00
        inet addr:192.168.1.100  Bcast:192.168.1.255  Mask:255.255.255.0
        inet6 addr: ffff::ffff:ffff:ffff:fff/64 Scope:Link
        UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
        RX packets:0 errors:0 dropped:0 overruns:0 frame:0
        TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
        collisions:0 txqueuelen:1000
        RX bytes:0 (0 GB)  TX bytes:0 (0 GB)
```

So I decided to develop something similar in PHP. This library is not exactly the same than python one. Python's sh allows more cool things such as non-blocking processes, baking, ... not available in my PHP one's, but at least I can call shell programs as functions in a simple way (and that's was my scope). Let's start.

One simple example of Process:

```php
use Symfony\Component\Process\Process;
 
$process = new Process('-latr ~');
$process->setTimeout(3600);
$process->run();
 
echo $process->getOutput();
```

With sh library we can do: 

```php
use Sh/Sh;
 
$sh  = new Sh();
echo $sh->ls('-latr ~');
```

You can check the source code in [github](https://github.com/gonzalo123/sh), but it's very simple one. Basically it's a [parser](https://github.com/gonzalo123/sh/blob/master/lib/Sh/Parser.php) that creates the command line string, and another [class](https://github.com/gonzalo123/sh/blob/master/lib/Sh/Sh.php) that calls to the parser and sends the output to Process component. Whit the magic function __call we can use shell commands as functions.

The command's arguments can be one string '-latr ~' or one array ['-latr', '~']. You can see more example in the unit tests [here](https://github.com/gonzalo123/sh/blob/master/tests/ParserTest.php)

Symfony/Process also allows us to get feedback in real time:

```php
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

**Sh** uses this feature, so we can do things like that: 
```php
$sh->tail('/var/log/messages', function ($buffer)  {
    echo $buffer;
});
```

We can see more examples here: \

```php
<?php
error_reporting(-1);
include __DIR__ . '/../vendor/autoload.php';
use Sh\Sh;
 
echo Sh::factory()->runCommnad('notify-send', ['-t', 5000, 'title', 'HOLA']);
$sh  = new Sh();
echo $sh->ifconfig("eth0");
echo $sh->ls('-latr ~');
echo $sh->ls(['-latr', '~']);
$sh->tail('-f /var/log/apache2/access.log', function ($buffer)  {
    echo $buffer;
});
```

As I said before the library is in [github](https://github.com/gonzalo123/sh) and also you can use with [composer](https://packagist.org/packages/gonzalo123/sh):

```commandline
require: "gonzalo123/sh": "dev-master"
```

**Updated!**

Now Sh library supports chained arguments (baking)

```php
// chainable commands (baking)
$sh->ssh(array('myserver.com', '-p' => 1393))->whoami();
// executes: ssh myserver.com -p 1393 whoami
 
$sh->ssh(array('myserver.com', '-p' => 1393))->tail(array("/var/log/dumb_daemon.log", 'n' => 100));
// executes: ssh myserver.com -p 1393 tail /var/log/dumb_daemon.log -n 100
});
```
