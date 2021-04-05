---
layout: post
title: "Playing with RabbitMQ (part 2). Now with Python"
date: "2017-03-13"
categories: 
  - "python"
  - "technology"
tags: 
  - "rabbitmq"
coverImage: "111590.png"
---

Do you remember the las post about [RabbitMQ](http://gonzalo123.com/2017/02/20/playing-with-rabbitmq-php-and-node/)? In that post we created a small wrapper library to use RabbitMQ with node and PHP. I also work with Python and I also want to use the same RabbitMQ wrapper here. With Python there're several libraries to use Rabbit. I'll use [pika](https://pika.readthedocs.io/en/0.10.0/).

The idea is the same than the another post. I want to use queues, exchanges and RPCs. So let's start with queues:

We can create a queue receiver called 'queue.backend' 

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
def onData(data):
    print data['aaa']
 
builder.queue('queue.backend', server).receive(onData)
```

and emit messages to the queue 

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
queue = builder.queue('queue.backend', server)
 
queue.emit({"aaa": 1})
queue.emit({"aaa": 2})
queue.emit({"aaa": 3})
```

The library (as the PHP and ones). Uses a builder class to create our instances 

```python
from queue import Queue
from rpc import RPC
from exchange import Exchange
 
defaults = {
    'queue': {
        'queue': {
            'passive': False,
            'durable': True,
            'exclusive': False,
            'autoDelete': False,
            'nowait': False
        },
        'consumer': {
            'noLocal': False,
            'noAck': False,
            'exclusive': False,
            'nowait': False
        }
    },
    'exchange': {
        'exchange': {
            'passive': False,
            'durable': True,
            'autoDelete': True,
            'internal': False,
            'nowait': False
        },
        'queue': {
            'passive': False,
            'durable': True,
            'exclusive': False,
            'autoDelete': True,
            'nowait': False
        },
        'consumer': {
            'noLocal': False,
            'noAck': False,
            'exclusive': False,
            'nowait': False
        }
    },
    'rpc': {
        'queue': {
            'passive': False,
            'durable': True,
            'exclusive': False,
            'autoDelete': True,
            'nowait': False
        },
        'consumer': {
            'noLocal': False,
            'noAck': False,
            'exclusive': False,
            'nowait': False
        }
    }
}
 
def queue(name, server):
    conf = defaults['queue']
    conf['server'] = server
 
    return Queue(name, conf)
 
def rpc(name, server):
    conf = defaults['rpc']
    conf['server'] = server
 
    return RPC(name, conf)
 
def exchange(name, server):
    conf = defaults['exchange']
    conf['server'] = server
 
    return Exchange(name, conf)
```

And our Queue class

```python
import pika
import json
import time
 
class Queue:
    def __init__(self, name, conf):
        self.name = name
        self.conf = conf
 
    def emit(self, data=None):
        credentials = pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port'], credentials=credentials))
        channel = connection.channel()
 
        queueConf = self.conf['queue']
        channel.queue_declare(queue=self.name, passive=queueConf['passive'], durable=queueConf['durable'], exclusive=queueConf['exclusive'], auto_delete=queueConf['autoDelete'])
 
        channel.basic_publish(exchange='', routing_key=self.name, body=json.dumps(data), properties=pika.BasicProperties(delivery_mode=2))
        connection.close()
 
    def receive(self, callback):
        credentials = pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port'], credentials=credentials))
        channel = connection.channel()
 
        queueConf = self.conf['queue']
        channel.queue_declare(queue=self.name, passive=queueConf['passive'], durable=queueConf['durable'], exclusive=queueConf['exclusive'], auto_delete=queueConf['autoDelete'])
 
        def _callback(ch, method, properties, body):
            callback(json.loads(body))
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print "%s %s::%s" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name, body)
 
        print "%s Queue '%s' initialized" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name)
        consumerConf = self.conf['consumer']
        channel.basic_qos(prefetch_count=1)
        channel.basic_consume(_callback, self.name, no_ack=consumerConf['noAck'], exclusive=consumerConf['exclusive'])
 
        channel.start_consuming()
```

We also want to use exchanges to emit messages without waiting for answers, just as a event broadcast. We can emit messages: 

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
exchange = builder.exchange('process.log', server)
 
exchange.emit("xxx.log", "aaaa")
exchange.emit("xxx.log", ["11", "aaaa"])
exchange.emit("yyy.log", "aaaa")
```

And listen to messages 

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
def onData(routingKey, data):
    print routingKey, data
 
builder.exchange('process.log', server).receive("yyy.log", onData)
```

That's the class 

```python
import pika
import json
import time
 
class Exchange:
    def __init__(self, name, conf):
        self.name = name
        self.conf = conf
 
    def emit(self, routingKey, data=None):
        credentials = pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port'], credentials=credentials))
        channel = connection.channel()
 
        exchangeConf = self.conf['exchange']
        channel.exchange_declare(exchange=self.name, type='topic', passive=exchangeConf['passive'], durable=exchangeConf['durable'], auto_delete=exchangeConf['autoDelete'], internal=exchangeConf['internal'])
        channel.basic_publish(exchange=self.name, routing_key=routingKey, body=json.dumps(data))
        connection.close()
 
    def receive(self, bindingKey, callback):
        credentials = pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port'], credentials=credentials))
        channel = connection.channel()
 
        exchangeConf = self.conf['exchange']
        channel.exchange_declare(exchange=self.name, type='topic', passive=exchangeConf['passive'], durable=exchangeConf['durable'], auto_delete=exchangeConf['autoDelete'], internal=exchangeConf['internal'])
 
        queueConf = self.conf['queue']
        result = channel.queue_declare(passive=queueConf['passive'], durable=queueConf['durable'], exclusive=queueConf['exclusive'], auto_delete=queueConf['autoDelete'])
        queue_name = result.method.queue
 
        channel.queue_bind(exchange=self.name, queue=queue_name, routing_key=bindingKey)
 
        print "%s Exchange '%s' initialized" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name)
 
        def _callback(ch, method, properties, body):
            callback(method.routing_key, json.loads(body))
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print "%s %s:::%s" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name, body)
 
        consumerConf = self.conf['consumer']
        channel.basic_consume(_callback, queue=queue_name, no_ack=consumerConf['noAck'], exclusive=consumerConf['exclusive'])
        channel.start_consuming()
```

And finally we can use RPCs. Emit \

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
print builder.rpc('rpc.hello', server).call("Gonzalo", "Ayuso")
```

And the server side 

```python
from rabbit import builder
 
server = {
    'host': 'localhost',
    'port': 5672,
    'user': 'guest',
    'pass': 'guest',
}
 
def onData(name, surname):
    return "Hello %s %s" % (name, surname)
 
builder.rpc('rpc.hello', server).server(onData)
```

And that's the class 

```python
import pika
import json
import time
import uuid
 
class RPC:
    def __init__(self, name, conf):
        self.name = name
        self.conf = conf
 
    def call(self, *params):
        pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port']))
        channel = connection.channel()
 
        queueConf = self.conf['queue']
        result = channel.queue_declare(queue='', passive=queueConf['passive'], durable=queueConf['durable'], exclusive=queueConf['exclusive'], auto_delete=queueConf['autoDelete'])
        callback_queue = result.method.queue
        consumerConf = self.conf['consumer']
        channel.basic_consume(self.on_call_response, no_ack=consumerConf['noAck'], exclusive=consumerConf['exclusive'], queue='')
 
        self.response = None
        self.corr_id = str(uuid.uuid4())
        channel.basic_publish(exchange='', routing_key=self.name, properties=pika.BasicProperties(reply_to=callback_queue, correlation_id=self.corr_id), body=json.dumps(params))
        while self.response is None:
            connection.process_data_events()
        return self.response
 
    def on_call_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body
 
    def server(self, callback):
        pika.PlainCredentials(self.conf['server']['user'], self.conf['server']['pass'])
        connection = pika.BlockingConnection(pika.ConnectionParameters(host=self.conf['server']['host'], port=self.conf['server']['port']))
        channel = connection.channel()
 
        queueConf = self.conf['queue']
        channel.queue_declare(self.name, passive=queueConf['passive'], durable=queueConf['durable'], exclusive=queueConf['exclusive'], auto_delete=queueConf['autoDelete'])
 
        channel.basic_qos(prefetch_count=1)
        consumerConf = self.conf['consumer']
 
        def on_server_request(ch, method, props, body):
            response = callback(*json.loads(body))
 
            ch.basic_publish(exchange='', routing_key=props.reply_to, properties=pika.BasicProperties(correlation_id=props.correlation_id), body=json.dumps(response))
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print "%s %s::req => '%s' response => '%s'" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name, body, response)
 
        channel.basic_consume(on_server_request, queue=self.name, no_ack=consumerConf['noAck'], exclusive=consumerConf['exclusive'])
 
        print "%s RPC '%s' initialized" % (time.strftime("[%d/%m/%Y-%H:%M:%S]", time.localtime(time.time())), self.name)
        channel.start_consuming()
```

And that's all. Full project is available within my [github](https://github.com/gonzalo123/RabbitMQ-python) account
