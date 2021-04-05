---
layout: post
title: "Upgrading Cordova-Android apps outside Google Play Store with angularjs"
date: "2014-07-28"
tags: 
  - "android"
  - "angularjs"
  - "php"
  - "technology"
  - "cordova"
  - "js"
  - "phonegap"
  - "silex"
---

Recent months I've working with enterprise mobile applications. This apps are't distributed using any marketplace, so I need to handle the distributions process. With Android you can compile your apps, create your APK files and distribute them. You can send the files by email, use a download link, send the file with bluetooth, or whatever. With iOS is a bit different. You need to purchase one Enterprise license, compile the app and distribute your IPA files using Apple's standards.

OK, but this post is not about how to distribute apps outside the markets. This post is about one big problem that appears when we need to upgrade our apps. How do the user knows that there's a new version of the application and he needs to upgrade? When we work inside Google Play Store we don't need to worry about it, but if we distribute our apps manually we need do something. We can send push notifications or email to the user to inform about the new version. Let me show you how I'm doing it.

My problem isn't only to let know to the user about a new version. Sometimes I also need to ensure that the user runs the last version of the app. Imagine a critical bug (solved in the last release) but the user don't upgrade.

First we need to create a static html page where the user can download the APK file. Imagine that this is the url where the user can download the last version of the app:

```commandline
http://192.168.1.1:8888/app.apk
```

We can check the version of the app against the server each time the user opens the application, but this check means network communication and it's slow. We need to reduce the communication between client and server to the smallest expression and only when it's strictly necessary. Check the version each time can be good in a desktop application, but it reduces the user experience with mobile apps. My approach is slightly different. Normally we use [token based authentication](http://gonzalo123.com/2014/06/09/token-based-authentication-with-silex-and-angularjs/ "Token based authentication with Silex and AngularJS") [within](http://gonzalo123.com/2014/05/05/token-based-authentication-with-silex-applications/ "Token based authentication with Silex Applications") mobile apps. That's means we need to send our token with all request. If we send the token, we also can send the version.

In a angular app we can define the version and the path of our apk using a key-value store.

```javascript
.value('config', {
        version: 4,
        androidAPK: "http://192.168.1.1:8888/app.apk"
    })
```

Now we need to add version parameter to each request (we can easily create a custom http service to append this parameter to each request automatically, indeed)

```javascript
$http.get('http://192.168.1.1:8888/api/doSomething', {params: {_version: config.version}})
    .success(function (data) {
        alert("OK");
    })
    .error(function (err, status) {
        switch (status) {
            case 410:
                $state.go('upgrade');
                break;
        }
    });
```

We can create a simple backend to take care of the version and throws an HTTP exception (one 410 HTTP error for example) if versions doesn't match. Here you can see a simple Silex example:

```php
<?php
 
include __DIR__ . "/../vendor/autoload.php";
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Exception\HttpException;
 
$app = new Application([
    'debug'   => true,
    'version' => 4,
]);
 
$app->after(function (Request $request, Response $response) {
    $response->headers->set('Access-Control-Allow-Origin', '*');
});
 
$app->get('/api/doSomething', function (Request $request, Application $app) {
    if ($request->get('_version') != $app['version']) {
        throw new HttpException(410, "Wrong version");
    } else {
        return $app->json('hello');
    }
});
 
$app->run();
```

As you can see we need to take care about [CORS](http://gonzalo123.com/2013/12/16/enabling-cors-in-a-restfull-silex-server-working-with-a-phonegapcordova-application/ "Enabling CORS in a RESTFull Silex server, working with a phonegap/cordova applications")

With this simple example we can realize if user has a wrong version within each server request. If version don't match we can, for example redirect to an specific route to inform that the user needs to upgrade the app and provide a link to perform the action.

With Android we cannot create a link to APK file. It doesn't work. We need to download the APK (using [FileTransfer](https://github.com/apache/cordova-plugin-file-transfer/blob/master/doc/index.md) plugin) and open the file using [webintent](https://github.com/Initsogar/cordova-webintent) plugin.

The code is very simple:

```javascript
var fileTransfer = new FileTransfer();
fileTransfer.download(encodeURI(androidUrl), 
    "cdvfile://localhost/temporary/app.apk",
    function (entry) {
        window.plugins.webintent.startActivity({
            action: window.plugins.webintent.ACTION_VIEW,
            url: entry.toURL(),
            type: 'application/vnd.android.package-archive'
        }, function () {
        }, function () {
            alert('Failed to open URL via Android Intent.');
            console.log("Failed to open URL via Android Intent. URL: " + entry.fullPath);
        });
    }, function (error) {
        console.log("download error source " + error.source);
        console.log("download error target " + error.target);
        console.log("upload error code" + error.code);
    }, true);
```

And basically that's all. When user self-upgrade the app it closes automatically and he needs to open it again, but now with the correct version.
