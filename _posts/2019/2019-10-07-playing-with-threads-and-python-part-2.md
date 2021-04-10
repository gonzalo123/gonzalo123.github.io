---
layout: post
title: "Playing with threads and Python. Part 2"
date: "2019-10-07"
categories: 
  - "python"
  - "rabbitmq"
tags: 
  - "python"
  - "technology"
  - "rabbitmq"
  - "threads"
coverImage: "119075.png"
---

Today I want to keep on playing with python and threads (part1 [here](http://gonzalo123.com/2019/08/19/playing-with-threads-and-python/)). The idea is create one simple script that prints one asterisk in the console each second. Simple, isn't it?

```python
from time import sleep
while True:
    print("*")
    sleep(1)
```

I want to keep this script running but I want to send one message externally, for example using RabbitMQ, and do something within the running script. In this demo, for example, stop the script.

In javascript we can do it with a single tread process using the setInterval function but, since the rabbit listener with pika is a blocking action, we need to use threads in Python (please tell me if I'm wrong). The idea is to create a circuit breaker condition in the main loop to check if I need to stop or not the main thread.

First I've created my Rabbit listener in a thread:

```python
from queue import Queue, Empty
import threading
import pika
import os
 
 
class Listener(threading.Thread):
    def __init__(self, queue=Queue()):
        super(Listener, self).__init__()
        self.queue = queue
        self.daemon = True
 
    def run(self):
        channel = self._get_channel()
        channel.queue_declare(queue='stop')
 
        channel.basic_consume(
            queue='stop',
            on_message_callback=lambda ch, method, properties, body: self.queue.put(item=True),
            auto_ack=True)
 
        channel.start_consuming()
 
    def stop(self):
        try:
            return True if self.queue.get(timeout=0.05) is True else False
        except Empty:
            return False
 
    def _get_channel(self):
        credentials = pika.PlainCredentials(
            username=os.getenv('RABBITMQ_USER'),
            password=os.getenv('RABBITMQ_PASS'))
 
        parameters = pika.ConnectionParameters(
            host=os.getenv('RABBITMQ_HOST'),
            credentials=credentials)
 
        connection = pika.BlockingConnection(parameters=parameters)
 
        return connection.channel()
```

Now in the main process I start the Listener and I enter in one endless loop to print my asterisk each second but at the end of each loop I check if I need to stop the process or not

```python
from Listener import Listener
from dotenv import load_dotenv
import logging
from time import sleep
import os
 
logging.basicConfig(level=logging.INFO)
 
current_dir = os.path.dirname(os.path.abspath(__file__))
load_dotenv(dotenv_path="{}/.env".format(current_dir))
 
l = Listener()
l.start()
 
 
def main():
    while True:
        logging.info("*")
        sleep(1)
        if l.stop():
            break
 
 
if __name__ == '__main__':
    main()
```

As we can see int the stop function we're using the queue.Queue package to communicate with our listener loop.

And that's all. In the example I also provide a minimal RabbitMQ server in a docker container.

```yaml
version: '3.4'
 
services:
  rabbit:
    image: rabbitmq:3-management
    restart: always
    environment:
      RABBITMQ_ERLANG_COOKIE:
      RABBITMQ_DEFAULT_VHOST: /
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS}
    ports:
      - "15672:15672"
      - "5672:5672"
```

Source code available in my [github](https://github.com/gonzalo123/threads2.py)
