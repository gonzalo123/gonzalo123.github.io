---
layout: post
title: "Building a AngularJS provider for hello.js library"
date: "2015-03-02"
tags: 
  - "angularjs"
  - "technology"
  - "angularjs"
  - "hello-js"
  - "javascript"
  - "js"
  - "oaut2"
---

This days I've been playing with [hello.js](http://adodson.com/hello.js/). Hello is a A client-side Javascript SDK for authenticating with OAuth2 web services. It's pretty straightforward to use and well explained at documentation. I want to use it within AngularJS projects. OK, I can include the library and use the global variable "hello", but it isn't cool. I want to create a reusable module and available with [Bower](http://bower.io/). Let's start.

Imagine one simple AngularJS application

```javascript
(function () {
    angular.module('G', [])
        .config(function ($stateProvider, $urlRouterProvider) {
            $urlRouterProvider.otherwise("/");
            $stateProvider
                .state('login', {
                    url: "/",
                    templateUrl: "partials/home.html",
                    controller: "LoginController"
                })
                .state('home', {
                    url: "/login",
                    template: "partials/home.html"
                });
        })
 
        .controller('LoginController', function ($scope) {
            $scope.login = function () {
            };
        })
})();
```

Now we can include our references within our bower.json file

```json
"dependencies": {
    "hello": "~1.4.1",
    "ng-hello": "*"
  }
```

and append those references to our index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="initial-scale=1, maximum-scale=1, user-scalable=no, width=device-width">
    <title>G</title>
 
    <script type="text/javascript" src="assets/hello/dist/hello.all.js"></script>
    <script type="text/javascript" src="assets/ng-hello/dist/ng-hello.js"></script>
    <script src="js/app.js"></script>
</head>
<body ng-app="G">
<div ui-view></div>
 
</body>
</html>
```

Our ng-hello is just a service provider that wraps hello.js 

```javascript
(function (hello) {
    angular.module('ngHello', [])
        .provider('hello', function () {
            this.$get = function () {
                return hello;
            };
 
            this.init = function (services, options) {
                hello.init(services, options);
            };
        });
})(hello);
```

That's means that we configure the service in config callback and in our run callback we can set up events

```javascript
(function () {
    angular.module('G', ['ngHello'])
        .config(function ($stateProvider, $urlRouterProvider, helloProvider) {
            helloProvider.init({
                twitter: 'myTwitterToken'
            });
 
            $urlRouterProvider.otherwise("/");
            $stateProvider
                .state('login', {
                    url: "/",
                    templateUrl: "partials/home.html",
                    controller: "LoginController"
                })
                .state('home', {
                    url: "/login",
                    template: "partials/home.html"
                });
        })
 
        .run(function ($ionicPlatform, $log, hello) {
            hello.on("auth.login", function (r) {
                $log.log(r.authResponse);
            });
        });
})();
```

And finally we can perform a twitter login within our controller

```javascript
(function () {
    angular.module('G')
        .controller('LoginController', function ($scope, hello) {
            $scope.login = function () {
                hello('twitter').login();
            };
        })
    ;
})();
```

And that's all. You can see the whole library in my github account [here](https://github.com/gonzalo123/ngHello)
