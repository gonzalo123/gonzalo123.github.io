---
layout: post
title: "Reading Modbus devices with Python from a PHP/Silex Application via Gearman worker"
date: "2016-04-18"
tags: 
  - "industrial-automation"
  - "php"
  - "python"
  - "technology"
  - "twig"
  - "automation"
  - "gearman"
  - "modbus"
  - "silex"
---

Yes. I know. I never know how to write a good tittle to my posts. Let me show one integration example that I've been working with this days. Let's start.

In industrial automation there're several standard protocols. [Modbus](https://en.wikipedia.org/wiki/Modbus) is one of them. Maybe isn't the coolest or the newest one (like [OPC](https://en.wikipedia.org/wiki/Open_Platform_Communications) or [OPC/UA](https://en.wikipedia.org/wiki/OPC_Unified_Architecture)), but we can speak Modbus with a huge number of devices.

I need to read from one of them, and show a couple of variables in a Web frontend. Imagine the following fake Modbus server (it emulates my real Modbus device)

```python
#!/usr/bin/env python
 
##
# Fake modbus server
# - exposes "Energy" 66706 = [1, 1170]
# - exposes "Power" 132242 = [2, 1170]
##
 
from pymodbus.datastore import ModbusSlaveContext, ModbusServerContext
from pymodbus.datastore import ModbusSequentialDataBlock
from pymodbus.server.async import StartTcpServer
import logging
 
logging.basicConfig()
log = logging.getLogger()
log.setLevel(logging.DEBUG)
 
hrData = [1, 1170, 2, 1170]
store = ModbusSlaveContext(hr=ModbusSequentialDataBlock(2, hrData))
 
context = ModbusServerContext(slaves=store, single=True)
 
StartTcpServer(context)
```

This server exposes two variables "Energy" and "Power". This is a fake server and it will returns always 66706 for energy and 132242 for power. Mobus is a binary protocol so 66706 = \[1, 1170\] and 132242 = \[2, 1170\]

I can read Modbus from PHP, but normally use Python for this kind of logic. I'm not going to re-write an existing logic to PHP. I'm not crazy enough. Furthermore my real Modbus device only accepts one active socket to retrieve information. That's means if two clients uses the frontend at the same time, it will crash. In this situations Queues are our friends.

I'll use a [Gearman](http://gearman.org/) worker (written in Python) to read Modbus information.

```python
from pyModbusTCP.client import ModbusClient
from gearman import GearmanWorker
import json
 
def reader(worker, job):
    c = ModbusClient(host="localhost", port=502)
 
    if not c.is_open() and not c.open():
        print("unable to connect to host")
 
    if c.is_open():
 
        holdingRegisters = c.read_holding_registers(1, 4)
 
        # Imagine we've "energy" value in position 1 with two words
        energy = (holdingRegisters[0] << 16) | holdingRegisters[1]
 
        # Imagine we've "power" value in position 3 with two words
        power = (holdingRegisters[2] << 16) | holdingRegisters[3]
 
        out = {"energy": energy, "power": power}
        return json.dumps(out)
    return None
 
worker = GearmanWorker(['127.0.0.1'])
 
worker.register_task('modbusReader', reader)
 
print 'working...'
worker.work()
```

Our backend is ready. Now we'll work with the frontend. In this example I'll use PHP and [Silex](http://silex.sensiolabs.org/).

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
use Silex\Application;
$app = new Application(['debug' => true]);
$app->register(new Silex\Provider\TwigServiceProvider(), array(
    'twig.path' => __DIR__.'/../views',
));
$app['modbusReader'] = $app->protect(function() {
    $client = new \GearmanClient();
    $client->addServer();
    $handle = $client->doNormal('modbusReader', 'modbusReader');
    $returnCode = $client->returnCode();
    if ($returnCode != \GEARMAN_SUCCESS) {
        throw new \Exception($this->client->error(), $returnCode);
    } else {
        return json_decode($handle, true);
    }
});
$app->get("/", function(Application $app) {
    return $app['twig']->render('home.twig', $app['modbusReader']());
});
$app->run();
```

As we can see the frontend is a simple Gearman client. It uses our Python worker to read information from Modbus and render a simple html with a Twig template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Demo</title>
</head>
<body>
    Energy: {{ energy }}
    Power: {{ power }}
</body>
</html>
```

And that's all. You can see the full example in my [github](https://github.com/gonzalo123/modbusExample) account
