---
layout: post
title: "Generating push notifications with Pushbullet and Silex"
date: "2015-08-24"
tags: 
  - "api"
  - "push-notifications"
  - "pushbullet"
  - "silex"
  - "php"
  - "technology"
---

Sometimes I need to send push notifications to mobile apps ([Android](http://gonzalo123.com/2013/08/05/sending-android-push-notifications-from-php-to-phonegap-applications/) or IOS). It's not difficult. Maybe it's a bit nightmare the first times, but when you understand the process, it's straightforward. Last days I discover a cool service called [PushBullet](https://www.pushbullet.com/). It allows us to install one client in our Android/IOS or even desktop computer, and send push notifications between them.

Pushbullet also has a good [API](https://docs.pushbullet.com/), and it allows us to automate our push notifications. I've play a little bit with the API and my Raspberry Pi - home server. It's really simple to integrate the API with our Silex backend and send push notifications to our registered devices.

I've created one small service provider to enclose the API. The idea is to use one Silex application like this

```php
use Silex\Application;
use PushSilex\Silex\Provider\PushbulletServiceProvider;
 
$app = new Application(['debug' => true]);
 
$myToken = include(__DIR__ . '/../conf/token.php');
 
$app->register(new PushbulletServiceProvider($myToken));
 
$app->get("/", function () {
    return "Usage: GET /note/{title}/{body}";
});
 
$app->get("/note/{title}/{body}", function (Application $app, $title, $body) {
    return $app->json($app['pushbullet.note']($title, $body));
});
 
$app->run();
```

As we can see we're using one service providers called PushbulletServiceProvider. This service provides us 'pushbullet.note' and allows to send push notifications. We only need to configure our Service Provider with our Pushbulled's token and that's all.

```php
<?php
namespace PushSilex\Silex\Provider;
use Silex\ServiceProviderInterface;
use Silex\Application;
class PushbulletServiceProvider implements ServiceProviderInterface
{
    private $accessToken;
    const URI = 'https://api.pushbullet.com/v2/pushes';
    const NOTE = 'note';
    public function __construct($accessToken)
    {
        $this->accessToken = $accessToken;
    }
    public function register(Application $app)
    {
        $app['pushbullet.note'] = $app->protect(function ($title, $body) {
            return $this->push(self::NOTE, $title, $body);
        });
    }
    private function push($type, $title, $body)
    {
        $data = [
            'type'  => $type,
            'title' => $title,
            'body'  => $body,
        ];
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL            => self::URI,
            CURLOPT_HTTPHEADER     => ['Content-Type' => 'application/json'],
            CURLOPT_CUSTOMREQUEST  => 'POST',
            CURLOPT_POSTFIELDS     => $data,
            CURLOPT_HTTPAUTH       => CURLAUTH_BASIC,
            CURLOPT_USERPWD        => $this->accessToken . ':',
            CURLOPT_RETURNTRANSFER => true
        ]);
        $out = curl_exec($ch);
        curl_close($ch);
 
        return json_decode($out);
    }
    public function boot(Application $app)
    {
    }
}
```

Normally I use [Guzzle](guzzle.readthedocs.org) to handle HTTP clients, but in this example I've created a raw curl connection.

You can see the project in my [github](https://github.com/gonzalo123/pushsilex) account
