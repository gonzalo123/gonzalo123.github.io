---
layout: post
title: "Microservice container with Guzzle"
date: "2015-06-29"
tags: 
  - "flask"
  - "microservices"
  - "silex"
  - "slim"
  - "php"
  - "python"
  - "symfony"
  - "technology"
---

This days I'm reading about [Microservices](http://martinfowler.com/articles/microservices.html). The idea is great. Instead of building a monolithic script using one language/framowork. We create isolated services and we build our application using those services (speaking HTTP between services and application).

That's means we'll have several microservices and we need to use them, and maybe sometimes change one service with another one. In this post I want to build one small container to handle those microservices. Similar idea than Dependency Injection Containers.

As we're going to speak HTTP, we need a HTTP client. We can build one using curl, but in PHP world we have [Guzzle](http://guzzle.readthedocs.org/en/latest/), a great HTTP client library. In fact Guzzle has something similar than the idea of this post: Guzzle services, but I want something more siple.

Imagine we have different services: One [Silex](http://silex.sensiolabs.org/) service (PHP + Silex) 

```php
use Silex\Application;
 
$app = new Application();
 
$app->get('/hello/{username}', function($username) {
    return "Hello {$username} from silex service";
});
 
$app->run();
```

Another PHP service. This one using [Slim](http://www.slimframework.com/) framework 

```php
use Slim\Slim;
 
$app = new Slim();
 
$app->get('/hello/:username', function ($username) {
    echo "Hello {$username} from slim service";
});
 
$app->run();
```

And finally one Python service using [Flask](http://flask.pocoo.org/) framework

```python
from flask import Flask, jsonify
app = Flask(__name__)
 
@app.route('/hello/<username>')
def show_user_profile(username):
    return "Hello %s from flask service" % username
 
if __name__ == "__main__":
    app.run(debug=True, host='0.0.0.0', port=5000)
```

Now, with our simple container we can use one service or another 

```php
use Symfony\Component\Config\FileLocator;
use MSIC\Loader\YamlFileLoader;
use MSIC\Container;
 
$container = new Container();
 
$ymlLoader = new YamlFileLoader($container, new FileLocator(__DIR__));
$ymlLoader->load('container.yml');
 
echo $container->getService('flaskServer')->get('/hello/Gonzalo')->getBody() . "\n";
echo $container->getService('silexServer')->get('/hello/Gonzalo')->getBody() . "\n";
echo $container->getService('slimServer')->get('/hello/Gonzalo')->getBody() . "\n";
```

And that's all. You can see the project in my [github](https://github.com/gonzalo123/msc) account.
