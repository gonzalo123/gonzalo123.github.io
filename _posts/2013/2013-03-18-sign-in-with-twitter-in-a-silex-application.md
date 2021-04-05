---
layout: post
title: "Sign-in with Twitter in a Silex application."
date: "2013-03-18"
tags: 
  - "php"
  - "technology"
  - "guzzle"
  - "oauth"
  - "silex"
  - "twitter"
---

I've working in a pet-project with Silex and I wanted to perform a Sign-in with Twitter. Implementing Sign in with Twitter is pretty straightforward and it's also well explained in the Twitter's developers [site](https://dev.twitter.com/docs/auth/implementing-sign-twitter). Now we only need to implement those HTTP client requests within PHP. We can create the REST client with [curl](http://php.net/manual/book.curl.php) but nowadays I prefer to use the great library called [Guzzle](http://guzzlephp.org/) to perform those kind of opperations. So let's start.

The idea is to create something reusable. I don't want to spend too much time including the Sign-in with Twitter in my proyects, so my first idea was to create a class with all the needed code and mount this class as group of Silex controllers (as it's defined [here](http://silex.sensiolabs.org/doc/organizing_controllers.html)). I also want to keep the class as standard as possible and avoiding the usage of any other external dependencies (except Guzzle)..

Imagine a simple Silex application: 

```php
<?php
// www/index.php
include __DIR__ . "/../vendor/autoload.php";
 
$app = new Silex\Application();
$app->get('/', function () {
    return 'Hello';
});
 
$app->run();
```

Now I want to use a Sign-in with Twitter, so I will change the application to: 

```php
<?php
include __DIR__ . "/../vendor/autoload.php";
 
$app = new Silex\Application();
$app->register(new Silex\Provider\SessionServiceProvider());
 
$consumerKey    = "***";
$consumerSecret = "***";
 
$twitterLoggin = new SilexTwitterLogin($app, 'twitter');
$twitterLoggin->setConsumerKey($consumerKey);
$twitterLoggin->setConsumerSecret($consumerSecret);
$twitterLoggin->registerOnLoggin(function () use ($app, $twitterLoggin) {
    $app['session']->set($twitterLoggin->getSessionId(), [
        'user_id'            => $twitterLoggin->getUserId(),
        'screen_name'        => $twitterLoggin->getScreenName(),
        'oauth_token'        => $twitterLoggin->getOauthToken(),
        'oauth_token_secret' => $twitterLoggin->getOauthTokenSecret()
    ]);
});
 
$twitterLoggin->mountOn('/login', function () {
    return '<a href="/login/requestToken">login</a>';
});
 
$app->get('/', function () use ($app){
    return 'Hello ' . $app['session']->get('twitter')['screen_name'];
});
 
$app->run();
```

The application will redirects all requests (without the correct session) to the route **"/login"**. The login page has a simple link to the route: **"/login/requestToken"** (we can create a fancy template with Twig if we want, indeed). This route redirects the request to Twitter’s login page and after a successful login it will redirects back to the route that we have defined within our Twitter application. The library assumes that this callback's url is **"/login/callbackUrl"**. All this default routes can be defined by the user using the proper setters of the class.

When the sign-in is finished the application will trigger the callback defined in **registerOnLoggin** function and will redirects to the route "/". This route (called internally **"redirectOnSuccess"**) is also customizable with a setter.

And that's all. Library available at [github](https://github.com/gonzalo123/SilexTwitterLogin) and [packagist](https://packagist.org/packages/gonzalo123/silex-twitter-login)

```json
{
    "require": {
        "gonzalo123/silex-twitter-login": "dev-master"
    }
}
```
