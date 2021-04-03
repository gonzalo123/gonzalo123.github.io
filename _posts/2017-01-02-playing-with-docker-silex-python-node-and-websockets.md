---
layout: post
title: "Playing with Docker, Silex, Python, Node and WebSockets"
date: "2017-01-02"
categories: 
  - "docker"
  - "js"
  - "node-js"
  - "php"
  - "python"
  - "socket-io"
  - "symfony"
  - "technology"
  - "twig"
  - "web-development"
  - "websockets"
tags: 
  - "node"
  - "silex"
  - "websockets"
---

I'm learning [Docker](https://www.docker.com/). In this post I want to share a little experiment that I have done. I know the code looks like over-engineering but it's just an excuse to build something with docker and containers. Let me explain it a little bit.

The idea is build a Time clock in the browser. Something like this:

![Clock](/assets/images/skitch.png)

Yes I know. We can do it only with js, css and html but we want to hack a little bit more. The idea is to create:

- A Silex/PHP frontend
- A WebSocket server with socket.io/node
- A Python script to obtain the current time

WebSocket server will open 2 ports: One port to serve webSockets (socket.io) and another one as a http server (express). Python script will get the current time and it'll send it to the webSocket server. Finally one frontend(silex) will be listening to WebSocket's event and it will render the current time.

That's the WebSocket server (with socket.io and express) 

```javascript
var
    express = require('express'),
    expressApp = express(),
    server = require('http').Server(expressApp),
    io = require('socket.io')(server, {origins: 'localhost:*'})
    ;
 
expressApp.get('/tic', function (req, res) {
    io.sockets.emit('time', req.query.time);
    res.json('OK');
});
 
expressApp.listen(6400, '0.0.0.0');
 
server.listen(8080);
```

That's our Python script

```python
from time import gmtime, strftime, sleep
import httplib2
 
h = httplib2.Http()
while True:
    (resp, content) = h.request("http://node:6400/tic?time=" + strftime("%H:%M:%S", gmtime()))
    sleep(1)
```

And our Silex frontend 

```php
use Silex\Application;
use Silex\Provider\TwigServiceProvider;
 
$app = new Application(['debug' => true]);
$app->register(new TwigServiceProvider(), [
    'twig.path' => __DIR__ . '/../views',
]);
 
$app->get("/", function (Application $app) {
    return $app['twig']->render('index.twig', []);
});
 
$app->run();
```

using this twig template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Docker example</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
    <link href="css/app.css" rel="stylesheet">
    <script src="https://oss.maxcdn.com/html5shiv/3.7.3/html5shiv.min.js"></script>
    <script src="https://oss.maxcdn.com/respond/1.4.2/respond.min.js"></script>
</head>
<body>
<div class="site-wrapper">
    <div class="site-wrapper-inner">
        <div class="cover-container">
            <div class="inner cover">
                <h1 class="cover-heading">
                    <div id="display">
                        display
                    </div>
                </h1>
            </div>
        </div>
    </div>
</div>
<script src="//localhost:8080/socket.io/socket.io.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js"></script>
<script>
var socket = io.connect('//localhost:8080');
 
$(function () {
    socket.on('time', function (data) {
        $('#display').html(data);
    });
});
</script>
</body>
```

The idea is to use one Docker container for each process. I like to have all the code in one place so all containers will share the same volume with source code.

First the node container (WebSocket server)

```dockerfile
FROM node:argon
 
RUN mkdir -p /mnt/src
WORKDIR /mnt/src/node
 
EXPOSE 8080 6400
```

Now the python container 

````dockerfile
FROM python:2
 
RUN pip install httplib2
 
RUN mkdir -p /mnt/src
WORKDIR /mnt/src/python
````

And finally Frontend contailer (apache2 with Ubuntu 16.04)

```dockerfile
FROM ubuntu:16.04
 
RUN locale-gen es_ES.UTF-8
RUN update-locale LANG=es_ES.UTF-8
ENV DEBIAN_FRONTEND=noninteractive
 
RUN apt-get update -y
RUN apt-get install --no-install-recommends -y apache2 php libapache2-mod-php
RUN apt-get clean -y
 
COPY ./apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf
 
RUN mkdir -p /mnt/src
 
RUN a2enmod rewrite
RUN a2enmod proxy
RUN a2enmod mpm_prefork
 
RUN chown -R www-data:www-data /mnt/src
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_PID_FILE /var/run/apache2/apache2.pid
ENV APACHE_SERVERADMIN admin@localhost
ENV APACHE_SERVERNAME localhost
 
EXPOSE 80
```

Now we've got the three containers but we want to use all together. We'll use a docker-compose.yml file. The web container will expose port 80 and node container 8080. Node container also opens 6400 but this port is an internal port. We don't need to access to this port outside. Only Python container needs to access to this port. Because of that 6400 is not mapped to any port in docker-compose

```yaml
version: '2'
 
services:
  web:
    image: gonzalo123/example_web
    container_name: example_web
    ports:
     - "80:80"
    restart: always
    depends_on:
      - node
    build:
      context: ./images/php
      dockerfile: Dockerfile
    entrypoint:
      - /usr/sbin/apache2
      - -D
      - FOREGROUND
    volumes:
     - ./src:/mnt/src
 
  node:
    image: gonzalo123/example_node
    container_name: example_node
    ports:
     - "8080:8080"
    restart: always
    build:
      context: ./images/node
      dockerfile: Dockerfile
    entrypoint:
      - npm
      - start
    volumes:
     - ./src:/mnt/src
 
  python:
      image: gonzalo123/example_python
      container_name: example_python
      restart: always
      depends_on:
        - node
      build:
        context: ./images/python
        dockerfile: Dockerfile
      entrypoint:
        - python
        - tic.py
      volumes:
       - ./src:/mnt/src
```

And that’s all. We only need to start our containers

```yaml
docker-compose up --build -d
```
and open our browser at: http://localhost to see our Time clock

Full source code available within my [github account](https://github.com/gonzalo123/time-clock-docker)
