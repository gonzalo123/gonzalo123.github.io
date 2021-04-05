---
layout: post
title: "Testing Phonegap/Cordova applications fast as hell in the device (with ionic framework)"
date: "2014-07-21"
tags: 
  - "angularjs"
  - "cordova"
  - "phonegap"
  - "technology"
  - "andoid"
  - "ionic"
  - "ios"
  - "js"
---

Normally when we work with Phonegap/[Cordova](http://cordova.apache.org/) applications we work in two phases. First we develop the application locally using our browser. That's "fast" phase. We change something within our code, then we reload our browser and we see the outcome. It isn't different from a "traditional" web developing process. Normally I use the [ionic framework](http://ionicframework.com/). Ionic is great and it also provides us a good tool to run a local server. We just type:

```commandline
ionic serve
```

And ionic starts a local server on port 8100 with our Cordova application, ready to test with the browser (it also opens the browser). That's not the cool part. Ionic also starts a live reload server at http://0.0.0.0:35729 and adds the following snippet at the end of our index.html

```html
<script type="text/javascript">//<![CDATA[
document.write('<script src="' + (location.protocol || 'http:') + '//' + (location.hostname || 'localhost') + ':35729/livereload.js?snipver=1" type="text/javascript"><\/script>')
//]]></script>
```

With this snippet our application will be reloaded when we add/remove something in our file tree (it runs a filesystem watcher in background).

But as I said before it's the "fast" phase and sooner or later we will need to run the application in the real device. OK we've got emulators, but they are horrible. Android emulator is incredible slow. IOS one is faster but we need to redeploy the application again and again with each change. For example when we correct a silly bug we need to run the following command to see the application running on the device:

```commandline
cordova run android --device
```

And it takes time (around 10 seconds). We've gone from the "fast" phase to the "slooooow" one. That means that I tried to avoid this phase until no remedy.

If you don't use plugins you can let this "slow" phase to the end, only to see the behaviour in the device and fix customizations, but ir we use plugins (camera plugin, push notifications or things like that) we really need to test on the real device. Those kind of things doesn't work in the browser or even with the emulator.

This "slow" phase droves me crazy, so I started to think a little bit about it. One Cordova app has two parts. The native one (java code in android and objective-c in ios) and the html/js part. We need to tell to our Cordova application where is the initial index.html. We usually do it in config.xml

```xml
<content src="index.html" />
```

But we can change this initial file and use a remote one. That's the way to create a "native" app from and existing web application.

```xml
<content src="http://gonzalo123.com" />
```

According to this we can start a local server in our host and use this local web server. Even in our LAN (if our android/ios device is the LAN of course)

```xml
<content src="http://192.168.1.1:8100/index.html" />
```

But, what happens with the plugins? Plugins needs cordova.js file and this file isn't in www folder. This file is generated when we build the application to a specific platform

```commandline
platforms/android/assets/www/cordova.js
```

So, what's the idea. The idea is:

- Run a local server with (inoic serve for example)
- Enable the fs watcher to restart the application when we change one file in the filesystem (inonic serve do it by default)
- Build the application and install it in the real device
- Use our local server to serve static files instead of build again and again the application with each change.

With this approach we only need to deploy the application to the real device when we want to add/remove one plugin. If we change anything in the static files (html, js, css) our app will be reloaded automatically. The "slow" phase turns into a "fast" phase.

How can we do it? It's easy. In this example I suppose that we're using one android device. If we use on iPhone we only need to change "adroid" to "ios".

First of all we need to prepare our index.html to enable auto-reload. "ionic serve" do it automatically but it thinks that we're going to use it with your host browser. Not with the "real" device. We can change it manually adding to our index.html (this snippet suppose that your host is 192.168.1.1 if it's a different one use your local IP address):

```html
<script type="text/javascript">//<![CDATA[
document.write('<script src="http://192.168.1.1:35729/livereload.js?snipver=1" type="text/javascript"><\/script>')
//]]></script>
```

Now we change our config.xml to use our local server instead of device's files:

```html
<!--<content src="index.html" />-->
<content src="http://192.168.1.105:8100/index.html" />
```

Now we need to deploy the application to our device:

```commandline
cordova run android --device
```

Each time we add/remove one plugin we need to redeploy to the device. But we need to keep in mind that our device will use the cordova.js from our local server, and not from its filesystem. "cordova run android --device" will generate the file to the platform and deploy them to the real device, but as well as we're going to use this file from our local server (in www), we need to create a set of symlinks in our www folder.

(I've got one setUp.sh file with this commands)

```commandline
cd www
ln -s ../platforms/android/assets/www/cordova.js
ln -s ../platforms/android/assets/www/cordova_plugins.js
ln -s ../platforms/android/assets/www/plugins
cd ..
```

Now can start the application's server in our host with:

```commandline
ionic serve --nobrowser
```

notice that we're using --nobrowser. We're using this parameter to not to open our local browser. We're going to use de device's Cordova's Webkit one, and also if we open our browser it will crash because cordova.js is present now and our local host isn't a real device.

Each time we need to redeploy the application to the device (new plugin for example) we need to remember to quit the symlinks, and redeploy.

(I've got one tearDown.sh file with this commands)

```commandline
rm www/cordova.js
rm www/cordova_plugins.js
rm -Rf www/plugins
```

And that's all. I now that this little hack may looks like something difficult but we need less than a minute to set up the environment and we will save thousand of seconds in the development process. I we work a little bit we can automate this process and turn it into a trivial operation, but at least now I feel very comfortable.

Of course you need to remember to clean the project when you finish and use the device's files. So we need to remove the auto-reload snippet in the index.html, remove symlinks and restore config.xml.

UPDATE:

ionic framework comes now with this feature out-of-the box. Just follow the instructions explained here: [link](http://ionicframework.com/blog/live-reload-all-things-ionic-cli/)
