---
layout: post
title: "Building a simple API proxy server with PHP"
date: "2012-08-13"
tags: 
  - "php"
  - "api"
  - "github"
  - "public-api"
  - "rest"
---

This days I'm playing with [Backbone](http://backbonejs.org/) and using public API as a source. The Web Browser has one horrible feature: It don't allow to fetch any external resource to our host due to the cross-origin restriction. For example if we have a server at localhost we cannot perform one AJAX request to another host different than localhost. Nowadays there is a header to allow it: [Access-Control-Allow-Origin](https://developer.mozilla.org/en-US/docs/HTTP_access_control). The problem is that the remote server must set up this header. For example I was playing with github's API and github doesn't have this header. If the server is my server, is pretty straightforward to put this header but obviously I'm not the sysadmin of github, so I cannot do it. What the solution? One possible solution is, for example, create a proxy server at localhost with PHP. With PHP we can use any remote API with curl (I wrote about it [here](http://gonzalo123.wordpress.com/2011/08/01/building-a-client-for-a-rest-api-with-php/) and [here](http://gonzalo123.wordpress.com/2010/02/06/building-a-rest-client-with-asynchronous-calls-using-php-and-curl/) for example). It's not difficult, but I asked myself: Can we create a dummy proxy server with PHP to handle any request to localhost and redirects to the real server, Instead of create one proxy for each request?. Let's start. Problably there is one open source solution (tell me if you know it) but I'm on holidays and I want to code a little bit (I now, it looks insane but that's me :) ).

The idea is: 

```php
...
$proxy->register('github', 'https://api.github.com');
...
```

And when I type:

> http://localhost/github/users/gonzalo123

and create a proxy to :

> https://api.github.com/users/gonzalo123

The request method is also important. If we create a POST request to localhost we want a POST request to github too.

This time we're not going to reinvent the wheel, so we will use symfony componets so we will use composer to start our project:

We create a conposer.json file with the dependencies: 

```json
{
    "require": {
        "symfony/class-loader":"dev-master",
        "symfony/http-foundation":"dev-master"
    }
}
```

Now

```commandline
php composer.phar install
```

And we can start coding. The script will look like this:

```php
register('github', 'https://api.github.com');
$proxy->run();
 
foreach($proxy->getHeaders() as $header) {
    header($header);
}
echo $proxy->getContent();
```

As we can see we can register as many servers as we want. In this example we only register github. The application only has two classes: [RestProxy](https://github.com/gonzalo123/rest-proxy/blob/master/lib/RestProxy/RestProxy.php), who extracts the information from the request object and calls to the real server through [CurlWrapper](https://github.com/gonzalo123/rest-proxy/blob/master/lib/RestProxy/CurlWrapper.php).

```php
<?php
namespace RestProxy;
 
class RestProxy
{
    private $request;
    private $curl;
    private $map;
 
    private $content;
    private $headers;
 
    public function __construct(\Symfony\Component\HttpFoundation\Request $request, CurlWrapper $curl)
    {
        $this->request  = $request;
        $this->curl = $curl;
    }
 
    public function register($name, $url)
    {
        $this->map[$name] = $url;
    }
 
    public function run()
    {
        foreach ($this->map as $name => $mapUrl) {
            return $this->dispatch($name, $mapUrl);
        }
    }
 
    private function dispatch($name, $mapUrl)
    {
        $url = $this->request->getPathInfo();
        if (strpos($url, $name) == 1) {
            $url         = $mapUrl . str_replace("/{$name}", NULL, $url);
            $queryString = $this->request->getQueryString();
 
            switch ($this->request->getMethod()) {
                case 'GET':
                    $this->content = $this->curl->doGet($url, $queryString);
                    break;
                case 'POST':
                    $this->content = $this->curl->doPost($url, $queryString);
                    break;
                case 'DELETE':
                    $this->content = $this->curl->doDelete($url, $queryString);
                    break;
                case 'PUT':
                    $this->content = $this->curl->doPut($url, $queryString);
                    break;
            }
            $this->headers = $this->curl->getHeaders();
        }
    }
 
    public function getHeaders()
    {
        return $this->headers;
    }
 
    public function getContent()
    {
        return $this->content;
    }
}
```

The RestProxy receive two instances in the constructor via dependency injection (CurlWrapper and Request). This architecture helps a lot in the [tests](https://github.com/gonzalo123/rest-proxy/blob/master/tests/SimpleTest.php), because we can mock both instances. Very helpfully when building RestProxy.

The RestProxy is registerd within [packaist](http://packagist.org/packages/gonzalo123/rest-proxy) so we can install it using composer installer:

First install componser

> curl -s https://getcomposer.org/installer | php

and create a new project:

> php composer.phar create-project gonzalo123/rest-proxy proxy

If we are using PHP5.4 (if not, what are you waiting for?) we can run the build-in server

```commandline
cd proxy
php -S localhost:8888 -t www/
```

Now we only need to open a web browser and type:

> http://localhost:8888/github/users/gonzalo123

The library is very minimal (it's enough for my experiment) and it does't allow authorization.

Of course full code is available in [github](https://github.com/gonzalo123/rest-proxy).
