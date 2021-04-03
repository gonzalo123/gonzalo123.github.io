---
layout: post
title: "Playing with RabbitMQ, PHP and node"
date: "2017-02-20"
categories: 
  - "js"
  - "node-js"
  - "php"
  - "technology"
tags: 
  - "javascrip"
  - "node"
  - "rabbitmq"
---

I need to use [RabbitMQ](https://www.rabbitmq.com/) in one project. I'm a big fan of [Gearman](https://gonzalo123.com/tag/gearman/), but I must admit Rabbit is much more powerful. In this project I need to handle with PHP code and node, so I want to build a wrapper for those two languages. I don't want to re-invent the wheel so I will use existing libraries ([php-amqplib](https://github.com/php-amqplib/php-amqplib) and [amqplib](https://github.com/squaremo/amqp.node) for node).

Basically I need to use three things: First I need to create exchange channels to log different actions. I need to decouple those actions from the main code. I also need to create work queues to ensure those works are executed. It doesn't matter if work is executed later but it must be executed. And finally RPC commands.

Let's start with the queues. I want to push events to a queue in PHP 

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
$queue = Builder::queue('queue.backend', $server);
$queue->emit(["aaa" => 1]);
```

and also with node 

```php
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var queue = builder.queue('queue.backend', server);
queue.emit({aaa: 1});
```

And I also want to register workers to those queues with PHP and node

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
Builder::queue('queue.backend', $server)->receive(function ($data) {
    error_log(json_encode($data));
});
```

```php
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var queue = builder.queue('queue.backend', server);
queue.receive(function (data) {
    console.log(data);
});
```

Both implementations use one builder. In this case we are using Queue: 

```php
namespace G\Rabbit;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
class Queue
{
    private $name;
    private $conf;
    public function __construct($name, $conf)
    {
        $this->name = $name;
        $this->conf = $conf;
    }
    private function createConnection()
    {
        $server = $this->conf['server'];
        return new AMQPStreamConnection($server['host'], $server['port'], $server['user'], $server['pass']);
    }
    private function declareQueue($channel)
    {
        $conf = $this->conf['queue'];
        $channel->queue_declare($this->name, $conf['passive'], $conf['durable'], $conf['exclusive'],
            $conf['auto_delete'], $conf['nowait']);
    }
    public function emit($data = null)
    {
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $this->declareQueue($channel);
        $msg = new AMQPMessage(json_encode($data),
            ['delivery_mode' => 2] # make message persistent
        );
        $channel->basic_publish($msg, '', $this->name);
        $channel->close();
        $connection->close();
    }
    public function receive(callable $callback)
    {
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $this->declareQueue($channel);
        $consumer = $this->conf['consumer'];
        if ($consumer['no_ack'] === false) {
            $channel->basic_qos(null, 1, null);
        }
        $channel->basic_consume($this->name, '', $consumer['no_local'], $consumer['no_ack'], $consumer['exclusive'],
            $consumer['nowait'],
            function ($msg) use ($callback) {
                call_user_func($callback, json_decode($msg->body, true), $this->name);
                $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
                $now = new \DateTime();
                echo '['.$now->format('d/m/Y H:i:s')."] {$this->name}::".$msg->body, "\n";
            });
        $now = new \DateTime();
        echo '['.$now->format('d/m/Y H:i:s')."] Queue '{$this->name}' initialized \n";
        while (count($channel->callbacks)) {
            $channel->wait();
        }
        $channel->close();
        $connection->close();
    }
}
```

```javascript
var amqp = require('amqplib/callback_api');
 
var Queue = function (name, conf) {
    return {
        emit: function (data, close=true) {
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    var msg = JSON.stringify(data);
 
                    ch.assertQueue(name, conf.queue);
                    ch.sendToQueue(name, new Buffer(msg));
                });
                if (close) {
                    setTimeout(function () {
                        conn.close();
                        process.exit(0)
                    }, 500);
                }
            });
        },
        receive: function (callback) {
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    ch.assertQueue(name, conf.queue);
                    console.log(new Date().toString() + ' Queue ' + name + ' initialized');
                    ch.consume(name, function (msg) {
                        console.log(new Date().toString() + " Received %s", msg.content.toString());
                        if (callback) {
                            callback(JSON.parse(msg.content.toString()), msg.fields.routingKey)
                        }
                        if (conf.consumer.noAck === false) {
                            ch.ack(msg);
                        }
                    }, conf.consumer);
                });
            });
        }
    };
};
 
module.exports = Queue;
```

We also want to emit messages using an exchange

Emiter:

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
$exchange = Builder::exchange('process.log', $server);
$exchange->emit("xxx.log", "aaaa");
$exchange->emit("xxx.log", ["11", "aaaa"]);
$exchange->emit("yyy.log", "aaaa");
```

````javascript
var builder = require('../../src/Builder');
 
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var exchange = builder.exchange('process.log', server);
 
exchange.emit("xxx.log", "aaaa");
exchange.emit("xxx.log", ["11", "aaaa"]);
exchange.emit("yyy.log", "aaaa");
````

and receiver: 

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
Builder::exchange('process.log', $server)->receive("yyy.log", function ($routingKey, $data) {
    error_log($routingKey." - ".json_encode($data));
});
```

```javascript
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var exchange = builder.exchange('process.log', server);
 
exchange.receive("yyy.log", function (routingKey, data) {
    console.log(routingKey, data);
});
```

And that's the PHP implementation: 

```php
namespace G\Rabbit;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
class Exchange
{
    private $name;
    private $conf;
    public function __construct($name, $conf)
    {
        $this->name = $name;
        $this->conf = $conf;
    }
    private function createConnection()
    {
        $server = $this->conf['server'];
        return new AMQPStreamConnection($server['host'], $server['port'], $server['user'], $server['pass']);
    }
    public function emit($routingKey, $data = null)
    {
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $conf = $this->conf['exchange'];
        $channel->exchange_declare($this->name, 'topic', $conf['passive'], $conf['durable'], $conf['auto_delete'],
            $conf['internal'], $conf['nowait']);
        $msg = new AMQPMessage(json_encode($data), [
            'delivery_mode' => 2, # make message persistent
        ]);
        $channel->basic_publish($msg, $this->name, $routingKey);
        $channel->close();
        $connection->close();
    }
    public function receive($bindingKey, callable $callback)
    {
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $conf = $this->conf['exchange'];
        $channel->exchange_declare($this->name, 'topic', $conf['passive'], $conf['durable'], $conf['auto_delete'],
            $conf['internal'], $conf['nowait']);
        $queueConf = $this->conf['queue'];
        list($queue_name, ,) = $channel->queue_declare("", $queueConf['passive'], $queueConf['durable'],
            $queueConf['exclusive'], $queueConf['auto_delete'], $queueConf['nowait']);
        $channel->queue_bind($queue_name, $this->name, $bindingKey);
        $consumerConf = $this->conf['consumer'];
        $channel->basic_consume($queue_name, '', $consumerConf['no_local'], $consumerConf['no_ack'],
            $consumerConf['exclusive'], $consumerConf['nowait'],
            function ($msg) use ($callback) {
                call_user_func($callback, $msg->delivery_info['routing_key'], json_decode($msg->body, true));
                $now = new \DateTime();
                echo '['.$now->format('d/m/Y H:i:s').'] '.$this->name.':'.$msg->delivery_info['routing_key'].'::', $msg->body, "\n";
                $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
            });
        $now = new \DateTime();
        echo '['.$now->format('d/m/Y H:i:s')."] Exchange '{$this->name}' initialized \n";
        while (count($channel->callbacks)) {
            $channel->wait();
        }
        $channel->close();
        $connection->close();
    }
}
```

And node: 

```javascript
var amqp = require('amqplib/callback_api');
 
var Exchange = function (name, conf) {
    return {
        emit: function (routingKey, data, close = true) {
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    var msg = JSON.stringify(data);
                    ch.assertExchange(name, 'topic', conf.exchange);
                    ch.publish(name, routingKey, new Buffer(msg));
                });
                if (close) {
                    setTimeout(function () {
                        conn.close();
                        process.exit(0)
                    }, 500);
                }
            });
        },
        receive: function (bindingKey, callback) {
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    ch.assertExchange(name, 'topic', conf.exchange);
                    console.log(new Date().toString() + ' Exchange ' + name + ' initialized');
                    ch.assertQueue('', conf.queue, function (err, q) {
 
                        ch.bindQueue(q.queue, name, bindingKey);
 
                        ch.consume(q.queue, function (msg) {
                            console.log(new Date().toString(), name, ":", msg.fields.routingKey, "::", msg.content.toString());
                            if (callback) {
                                callback(msg.fields.routingKey, JSON.parse(msg.content.toString()))
                            }
                            if (conf.consumer.noAck === false) {
                                ch.ack(msg);
                            }
                        }, conf.consumer);
                    });
                });
            });
        }
    };
};
 
module.exports = Exchange;
```

Finally we want to use RPC commands. In fact RPC implementations is similar than Queue but in this case client will receive an answer.

Client side

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
echo Builder::rpc('rpc.hello', $server)->call("Gonzalo", "Ayuso");
```

```javascript
var builder = require('../../src/Builder');
 
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var rpc = builder.rpc('rpc.hello', server);
rpc.call("Gonzalo", "Ayuso", function (data) {
    console.log(data);
});
```

Server side:

```php
use G\Rabbit\Builder;
$server = [
    'host' => 'localhost',
    'port' => 5672,
    'user' => 'guest',
    'pass' => 'guest',
];
Builder::rpc('rpc.hello', $server)->server(function ($name, $surname) use ($server) {
    return "Hello {$name} {$surname}";
});
```

```javascript
var builder = require('../../src/Builder');
 
var server = {
    host: 'localhost',
    port: 5672,
    user: 'guest',
    pass: 'guest'
};
 
var rpc = builder.rpc('rpc.hello', server);
 
rpc.server(function (name, surname) {
    return "Hello " + name + " " + surname;
});
```

And Implementations:

```php
namespace G\Rabbit;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
class RPC
{
    private $name;
    private $conf;
    public function __construct($name, $conf)
    {
        $this->name = $name;
        $this->conf = $conf;
    }
    private function createConnection()
    {
        $server = $this->conf['server'];
        return new AMQPStreamConnection($server['host'], $server['port'], $server['user'], $server['pass']);
    }
    public function call()
    {
        $params = (array)func_get_args();
        $response = null;
        $corr_id = uniqid();
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $queueConf = $this->conf['queue'];
        list($callback_queue, ,) = $channel->queue_declare("", $queueConf['passive'], $queueConf['durable'],
            $queueConf['exclusive'], $queueConf['auto_delete'], $queueConf['nowait']);
        $consumerConf = $this->conf['consumer'];
        $channel->basic_consume($callback_queue, '', $consumerConf['no_local'], $consumerConf['no_ack'],
            $consumerConf['exclusive'], $consumerConf['nowait'], function ($rep) use (&$corr_id, &$response) {
                if ($rep->get('correlation_id') == $corr_id) {
                    $response = $rep->body;
                }
            });
        $msg = new AMQPMessage(json_encode($params), [
            'correlation_id' => $corr_id,
            'reply_to'       => $callback_queue,
        ]);
        $channel->basic_publish($msg, '', $this->name);
        while (!$response) {
            $channel->wait();
        }
        return json_decode($response, true);
    }
    public function server(callable $callback)
    {
        $connection = $this->createConnection();
        $channel = $connection->channel();
        $queueConf = $this->conf['queue'];
        $channel->queue_declare($this->name, $queueConf['passive'], $queueConf['durable'], $queueConf['exclusive'],
            $queueConf['auto_delete'], $queueConf['nowait']);
        $now = new \DateTime();
        echo '['.$now->format('d/m/Y H:i:s')."] RPC server '{$this->name}' initialized \n";
        $channel->basic_qos(null, 1, null);
        $consumerConf = $this->conf['consumer'];
        $channel->basic_consume($this->name, '', $consumerConf['no_local'], $consumerConf['no_ack'],
            $consumerConf['exclusive'],
            $consumerConf['nowait'], function ($req) use ($callback) {
                $response = json_encode(call_user_func_array($callback, array_values(json_decode($req->body, true))));
                $msg = new AMQPMessage($response, [
                    'correlation_id' => $req->get('correlation_id'),
                    'delivery_mode'  => 2, # make message persistent
                ]);
                $req->delivery_info['channel']->basic_publish($msg, '', $req->get('reply_to'));
                $req->delivery_info['channel']->basic_ack($req->delivery_info['delivery_tag']);
                $now = new \DateTime();
                echo '['.$now->format('d/m/Y H:i:s').'] '.$this->name.":: req => '{$req->body}' response=> '{$response}'\n";
            });
        while (count($channel->callbacks)) {
            $channel->wait();
        }
        $channel->close();
        $connection->close();
    }
}
```

```javascript
var amqp = require('amqplib/callback_api');
 
var RPC = function (name, conf) {
    var generateUuid = function () {
        return Math.random().toString() +
            Math.random().toString() +
            Math.random().toString();
    };
 
    return {
        call: function () {
            var params = [];
            for (i = 0; i < arguments.length - 1; i++) {
                params.push(arguments[i]);
            }
            var callback = arguments[arguments.length - 1];
 
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    ch.assertQueue('', conf.queue, function (err, q) {
                        var corr = generateUuid();
 
                        ch.consume(q.queue, function (msg) {
                            if (msg.properties.correlationId == corr) {
                                callback(JSON.parse(msg.content.toString()));
                                setTimeout(function () {
                                    conn.close();
                                    process.exit(0)
                                }, 500);
                            }
                        }, conf.consumer);
                        ch.sendToQueue(name,
                            new Buffer(JSON.stringify(params)),
                            {correlationId: corr, replyTo: q.queue});
                    });
                });
            });
        },
        server: function (callback) {
            amqp.connect(`amqp://${conf.server.user}:${conf.server.pass}@${conf.server.host}:${conf.server.port}`, function (err, conn) {
                conn.createChannel(function (err, ch) {
                    ch.assertQueue(name, conf.queue);
                    console.log(new Date().toString() + ' RPC ' + name + ' initialized');
                    ch.prefetch(1);
                    ch.consume(name, function reply(msg) {
                        console.log(new Date().toString(), msg.fields.routingKey, " :: ", msg.content.toString());
                        var response = JSON.stringify(callback.apply(this, JSON.parse(msg.content.toString())));
                        ch.sendToQueue(msg.properties.replyTo,
                            new Buffer(response),
                            {correlationId: msg.properties.correlationId});
 
                        ch.ack(msg);
 
                    }, conf.consumer);
                });
            });
        }
    };
};
 
module.exports = RPC;
```

You can see whole projects at github: [RabbitMQ-php](https://github.com/gonzalo123/RabbitMQ-php), [RabbitMQ-node](https://github.com/gonzalo123/RabbitMQ-node)
