---
layout: post
title: "Enabling CORS in a RESTFull Silex server, working with a phonegap/cordova applications"
date: "2013-12-16"
tags: 
  - "cordova"
  - "phonegap"
  - "symfony"
  - "cors"
  - "php"
  - "silex"
---

This days I'm working with [phonegap](http://phonegap.com/)/[cordova](http://cordova.apache.org/) projects. I'm using [topcoat](http://topcoat.io/) and [AngularJs](http://angularjs.org/) to build the client side and [Silex](http://silex.sensiolabs.org/) for the backend. Cordova applications are "diferent" than a common web application. Our client side is normally located inside our mobile device (it's also possible to use remote webviews). Our cordova application must speak with our backend. The easiest way to perform this operation is to use a [REST](http://en.wikipedia.org/wiki/Representational_state_transfer). AngularJS has a great tool to connect with RESTFull [resources](http://docs.angularjs.org/api/ngResource.$resource). Silex is also great to build RESTFull services. I wrote a couple of posts about [it](http://gonzalo123.com/2013/06/03/working-with-angularjs-and-silex-as-resource-provider/).

With the first request form our AngularJS application (into our android/iphone device) to our Silex application, we will face with [CORS](http://www.w3.org/TR/cors/). We cannot perform a request from our "local" phonegap/cordova application to our remote WebServer. We cannot do it if we don't allow it explictily. With Silex it's pretty straight forward to do it. We can use the event dispatcher and change the request with after handler.

```php
$app->after(function (Request $request, Response $response) {
    $response->headers->set('Access-Control-Allow-Origin', '*');
});
```

We can do more strict, setting also "Access-Control-Allow-Methods" and "Access-Control-Allow-Headers" headers but only with this header we can work properly with our RESTFull Silex application from our phonegap/cordova application.
