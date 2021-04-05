---
layout: post
title: "Working with Ionic and PHP Backends. Remote debugging with PHP7 and Xdebug working with real devices"
date: "2015-12-14"
tags: 
  - "angularjs"
  - "cordova"
  - "ionic"
  - "js"
  - "phonegap"
  - "phpstorm"
  - "xdebug"
  - "angularjs"
  - "php"
  - "technology"
---

Sometimes I speak with PHP developers and they don't use remote debugging in their development environments. Some people don't like to use remote debugging. They prefer to use TDD and rely on the unit tests. That's a good point of view, but sometimes they don't use remote debugging only because they don't know how to do it, and that's inadmissible. [Remote debugger](http://xdebug.org) is a powerful tool especially to handle with legacy applications. I've using xdebug for years with my linux workstation for years. This days I'm using Mac and it's also very simple to set up xdebug here.

First we need to install PHP:

```commandline
brew install php70
```

Then Xdebug 

```commandline
brew install php70-xdebug
```

(in a Ubuntu box we only need to use apt-get instead of brew)

Now we need to setup xdebug to enable remote debugging: In a standard installation xdebug configuration is located at: /usr/local/etc/php/7.0/conf.d/ext-xdebug.ini

```ini
[xdebug]
zend_extension="/usr/local/opt/php70-xdebug/xdebug.so"
 
xdebug.remote_enable=1
xdebug.remote_port=9000
xdebug.profiler_enable=0
xdebug.profiler_output_dir="/tmp"
xdebug.idekey= "PHPSTORM"
xdebug.remote_connect_back = 1
xdebug.max_nesting_level = 250
```

And basically that's all. To set/unset the cookie you can use one bookmarklet in your browser (you can generate your bookmarklets [here](http://www.jetbrains.com/phpstorm/marklets/)). Or use a Chrome [extension](https://chrome.google.com/webstore/detail/xdebug-helper/eadndfjplgieldjbigjakmdgkmoaaaoc) to enable xdebug.

Now se only need to start the built-in server with

```commandline
php -S 0.0.0.0:8080
```

And remote debugging will be available Remote debugger works this way:

- We open on port within our IDE. In my case PHPStorm (it happens when we click on "Start listening for PHP debug connections")
- We set one cookie in our browser (it happens when click on Chrome extension)
- When our server receives one request with the cookie, it connects to the port that our IDE opens (usually port 9000). If you use a personal firewall in your workstation, ensure that you allow incoming connections to this port.

[![youtube](https://img.youtube.com/vi/hui9GKJGb8I/0.jpg)](https://www.youtube.com/watch?v=hui9GKJGb8I)


Nowadays I'm involved with several projects building hybrid applications with [Apache Cordova](https://cordova.apache.org/). In the Frontend I'm using [ionic](http://ionicframework.com/) and [Silex](http://silex.sensiolabs.org/) in the Backend. When I'm working with hybrid applications normally I go through two phases.

In the first one I build a working prototype. To to this I run a local server and I use my browser to develop the application. This phase is very similar than a traditional Web development process. If we also set up properly [LiveReload](https://chrome.google.com/webstore/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei), our application will be reloaded each time we change one javaScript file. Ionic framework integrates LiveReload and we only need to run:

```commandline
ionic serve -l
```

to start our application. We also need to start our backend server. For example

```commandline
php -S 0.0.0.0:8080 -t api/www
```

Now we can debug our Backend with remote debugger and Frontend with Chrome's developer's tools. Chrome also allows us to edit Frontend files and save them within the filesystem using [workspaces](https://developers.google.com/web/tools/setup/). This phase is the easy one. But sooner or later we'll need start working with a real device. We need a real device basically if we use plugins such as Camera plugin, Geolocation plugin, or things like that. OK there are emulators, but usually emulators don't allow to use all plugins in the same way than we use then with a real device. Chrome also allow us to see the console logs of the device from our workstation. OK we can see all logs of our plugged Android device using "adb logcat" but follow the flow of our logs with logcat is similar than understand Matrix code. It's a mess.

If we plug our android device to our computer and we open with Chrome: 

```commandline
chrome://inspect/#devices
```

We can see our device's console, use breakpoints and things like that. Cool, isn't it? Of course it only works if we compile our application without "--release" option. We can do something similar with Safary and iOS devices.

With ionic if we want to use LiveReload from the real device and not to recompile and re-install again and again our application each time we change our javaScript files, we can run the application using

```commandline
ionic run android --device -l
```

When we're developing our application and we're in this phase we also need to handle with [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing). CORS isn't a problem when we run our hybrid application in production. When we run the hybrid application with our device our "origin" is the local filesystem. That's means CORS don't apply, but when we run our application in the device, but served from our computer (when we use "-l" option), our origin isn't local filesystem. So if our Backend is served from another origin we need to enable CORS.

We can enable CORS in the backend. I've written about it [here](http://gonzalo123.com/2013/12/16/enabling-cors-in-a-restfull-silex-server-working-with-a-phonegapcordova-application/), but ionic people allows us a easier way. We can set up a local proxy to serve our backend through the same origin than the application does and forget about CORS. Here we can read a good [article](http://blog.ionic.io/handling-cors-issues-in-ionic/) about it.

Anyway if we want to start the remote debugger we need to create one cookie called XDEBUG\_SESSION. In the browser we can use chrome extension, but when we inspect the plugged device isn't so simple. It would be cool that ionic people allows us to inject cookies to our proxy server. I've try to see how to do it with ionic-cli. Maybe is possible but I didn't realize how to do it. Because of that I've created a simple AngularJS service to inject this cookie. Then, if I start listening debug connections in my IDE I'll be able to use remote debugger as well as I do when I work with the browser.

First we need to install service via Bower:

```commandline
bower install ng-xdebugger --save
```

Now we need to include javaScript files 

```html
<script src="lib/angular-cookies/angular-cookies.min.js"></script>
<script src="lib/ng-xdebugger/dist/gonzalo123.xdebugger.min.js"></script>
```

then we add our service to the project. 

```javascript
angular.module("starter", ["ionic", "gonzalo123.xdebugger"])
```

Now we only need to configure our application and set de debugger key (it must be the same key than we use within the server-side configuration of xdebug) 

```javascript
.config(function (xdebuggerProvider) {
        xdebuggerProvider.setKey('PHPSTORM');
    })
})
```

And that's all. The service is very simple. It only uses one http interceptor to inject the cookie in our http requests: 

```javascript
(function () {
    "use strict";
 
    angular.module("gonzalo123.xdebugger", ["ngCookies"])
        .provider("xdebugger", ['$httpProvider', function ($httpProvider) {
            var debugKey;
 
            this.$get = function () {
                return {
                    getDebugKey: function () {
                        return debugKey;
                    }
                };
            };
 
            this.setKey = function (string) {
                if (string) {
                    debugKey = string;
                    $httpProvider.interceptors.push("xdebuggerCookieInterceptor");
                }
            };
        }])
 
        .factory("xdebuggerCookieInterceptor", ['$cookieStore', 'xdebugger', function ($cookieStore, xdebugger) {
            return {
                response: function (response) {
                    $cookieStore.put("XDEBUG_SESSION", xdebugger.getDebugKey());
 
                    return response;
                }
            };
        }])
    ;
})();
```

And of course you can see the whole project in my [github](https://github.com/gonzalo123/ngXdebugger) account.
