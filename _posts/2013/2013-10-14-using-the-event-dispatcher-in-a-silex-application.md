---
layout: post
title: "Using the event dispatcher in a Silex application"
date: "2013-10-14"
tags: 
  - "symfony"
  - "symfony2"
  - "technology"
  - "php"
  - "silex"
---

Symfony has one component called [The Event Dispatcher](http://symfony.com/doc/current/components/event_dispatcher/introduction.html). This component is one implementation of [Mediator](http://en.wikipedia.org/wiki/Mediator_pattern) pattern and it's widely used in modern frameworks, such as Symfony. Silex, as a part of Symfony, also uses this component and we can easily use it in our projects. Let me show you one little example. Imagine one simple route in Silex to create one png file containing one text:

```php
$app->get("/gd/{text}", function($text) {
    $path = "/tmp/qr.png." . uniqid();
    $im = imagecreate(90, 30);
    $background = imagecolorallocate($im, 255, 255, 255);
    $color = imagecolorallocate($im, 0, 0, 0);
    imagestring($im, 5, 5, 5,  $text, $color);
    imagepng($im, $path);
    imagedestroy($im);
    return $app->sendFile($path);
});
```

It works, but there's one mistake. We need to unlink our temporally file _$path_, but where? We need do if after "return $app->sendFile($path);" but it's not possible.

```php
$app->get("/gd/{text}", function($text) {
    $path = "/tmp/qr.png." . uniqid();
    $im = imagecreate(90, 30);
    $background = imagecolorallocate($im, 255, 255, 255);
    $color = imagecolorallocate($im, 0, 0, 0);
    imagestring($im, 5, 5, 5,  $text, $color);
    imagepng($im, $path);
    imagedestroy($im);
    return $app->sendFile($path, 200, ['Content-Type' => 'image/png']);;
    unlink($path); // unreachable code
});
```

We can use BinaryFileResponse instead of the helper function "sendFile", but there's one smarter solution: The event dispatcher.

```php
use Symfony\Component\HttpKernel\KernelEvents;
 
$app->get("/gd/{text}", function($text) use (app) {
    $im = imagecreate(90, 30);
    $path = "/tmp/qr.png." . uniqid();
    $background = imagecolorallocate($im, 255, 255, 255);
    $color = imagecolorallocate($im, 0, 0, 0);
    imagestring($im, 5, 5, 5,  $text, $color);
    imagepng($im, $path);
    imagedestroy($im);
     
    $app['dispatcher']->addListener(KernelEvents::TERMINATE, function() use ($path) {
        unlink($path);
    });
 
    return $app->sendFile($path, 200, ['Content-Type' => 'image/png']);
});
```

_(Updated! thanks to [Hakin's](http://gonzalo123.com/2013/10/14/using-the-event-dispatcher-in-a-silex-application/#comment-5615) recommendation)_ Or even better using Silex's Filters. In this case we after or finish. In fact those filters are nothing more than an elegant way to speak to the event dispatcher.

```php
$app->get("/gd/{text}", function($text) use (app) {
    $im = imagecreate(90, 30);
    $path = "/tmp/qr.png." . uniqid();
    $background = imagecolorallocate($im, 255, 255, 255);
    $color = imagecolorallocate($im, 0, 0, 0);
    imagestring($im, 5, 5, 5,  $text, $color);
    imagepng($im, $path);
    imagedestroy($im);
     
    $app->after(function() use ($path) {
        unlink($path);
    });
 
    return $app->sendFile($path, 200, ['Content-Type' => 'image/png']);
});
```

We also can use the generic function to add events to the event listener: 

```php
use Symfony\Component\HttpKernel\KernelEvents;
 
$app->get("/gd/{text}", function($text) use (app) {
    $im = imagecreate(90, 30);
    $path = "/tmp/qr.png." . uniqid();
    $background = imagecolorallocate($im, 255, 255, 255);
    $color = imagecolorallocate($im, 0, 0, 0);
    imagestring($im, 5, 5, 5,  $text, $color);
    imagepng($im, $path);
    imagedestroy($im);
     
    $app->on(KernelEvents::TERMINATE, function() use ($path) {
        unlink($path);
    });
 
    return $app->sendFile($path, 200, ['Content-Type' => 'image/png']);
});
```

Now our temporally file will be deleted once a response is sent. Life is simpler with event dispatcher :)
