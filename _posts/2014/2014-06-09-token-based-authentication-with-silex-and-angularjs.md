---
layout: post
title: "Token based authentication with Silex and AngularJS"
date: "2014-06-09"
tags: 
  - "js"
  - "php"
  - "silex"
  - "topcoat"
  - "angularjs"
  - "cordova"
  - "phonegap"
  - "symfony"
---

According to my last [post](http://gonzalo123.com/2014/05/05/token-based-authentication-with-silex-applications/ "Token based authentication with Silex Applications") today we're going to create a AngularJS application that uses the Silex Backend that we create previously. The idea of this application is to use it within a Phonegap/Cordova application running in a mobile device.

The application will be show a login form if device haven't a correct token.

![](/assets/images/gonzalo_login_example_and_loginserviceprovider_php_-_token_-____work_projects_token_.jpg)

And with a correct token:

![](/assets/images/gonzalo_login_example.jpg)

Nothing new under the sun, isn't it?

Our front-end application will use [AngularJS](https://angularjs.org/) and [Topcoat](http://topcoat.io/).

```html
<!DOCTYPE html>
<html xmlns:ng="http://angularjs.org" lang="es" ng-app="G">
<head>
    <meta charset="utf-8"/>
    <meta name="format-detection" content="telephone=no"/>
    <!-- WARNING: for iOS 7, remove the width=device-width and height=device-height attributes. See https://issues.apache.org/jira/browse/CB-4323 -->
    <meta name="viewport"
          content="user-scalable=no, initial-scale=1, maximum-scale=1, minimum-scale=1, width=device-width, height=device-height, target-densitydpi=device-dpi"/>
    <link rel="stylesheet" type="text/css" href="/bower_components/topcoat/css/topcoat-mobile-light.min.css">
    <title>Gonzalo Login Example</title>
</head>
<body ng-controller="MainController">
 
<div ng-view class="main-content"></div>
 
<script src="/bower_components/angular/angular.min.js"></script>
<script src="/bower_components/angular-route/angular-route.min.js"></script>
 
<script src="js/app.js"></script>
<script src="js/services.js"></script>
 
</body>
</html>
```

And our AngularJS application:

```javascript
'use strict';
var appControllers, G;
var host = 'http://localhost:8080'; // server API url
 
appControllers = angular.module('appControllers', []);
G = angular.module('G', ['ngRoute', 'appControllers']);
 
G.run(function (httpG) {
    httpG.setHost(host);
});
 
G.config(['$routeProvider', function ($routeProvider) {
    $routeProvider.
        when('/login', {templateUrl: 'partials/login.html', controller: 'LoginController'}).
        when('/home', {templateUrl: 'partials/home.html', controller: 'HomeController'});
}]);
 
appControllers.controller('HomeController', ['$scope', 'httpG', '$location', function ($scope, httpG, $location) {
    $scope.hello = function () {
        httpG.get('/api/info').success(function (data) {
            if (data.status) {
                alert("Hello " + data.info.name + " " + data.info.surname);
            }
        });
    };
 
    $scope.logOut = function () {
        alert("Good bye!");
        httpG.removeToken();
        $scope.isAuthenticated = false;
        $location.path('login');
    };
}]);
 
appControllers.controller('MainController', ['$scope', '$location', 'httpG', function ($scope, $location, httpG) {
    $scope.isAuthenticated = false;
 
    if (httpG.getToken()) {
        $scope.isAuthenticated = true;
        $location.path('home');
    } else {
        $location.path('login');
    }
}]);
 
 
appControllers.controller('LoginController', ['$scope', '$location', 'httpG', function ($scope, $location, httpG) {
    $scope.user = {};
 
    $scope.doLogIn = function () {
        httpG.get('/auth/validateCredentials', {user: $scope.user.username, pass: $scope.user.password}).success(function (data) {
            if (data.status) {
                httpG.setToken(data.info.token);
                $scope.isAuthenticated = true;
                $location.path('home');
            } else {
                alert("login error");
            }
        }).error(function (error) {
            alert("Login Error!");
        });
    };
 
    $scope.doLogOut = function () {
        httpG.removeToken();
    };
}]);
```

In this example I'm using angular-route to handle the application's routes. Nowadays I'm swaping to angular-ui-router, but this example I'm still using "old-style" routes. We define two partials:

partial/home.html 

```html
<div class="topcoat-button-bar full" style="position: fixed; bottom: 0px;">
    <label class="topcoat-button-bar__item">
        <button class="topcoat-button full" ng-click="logOut()">
            <span class="">Logout</span>
        </button>
    </label>
    <label class="topcoat-button-bar__item">
        <button class="topcoat-button--cta full" ng-click="hello()">
            <span class="">Hello</span>
        </button>
    </label>
</div>
```

partial/login.html 

```html
<div class="topcoat-navigation-bar">
    <div class="topcoat-navigation-bar__item center full">
        <h1 class="topcoat-navigation-bar__title">Login</h1>
    </div>
</div>
 
<ul class="topcoat-list__container">
    <li class="topcoat-list__item center">
        <input ng-model="user.username" class="topcoat-text-input--large" type="text" name="user"
               placeholder="Username"/>
    </li>
    <li class="topcoat-list__item center">
        <input ng-model="user.password" class="topcoat-text-input--large" type="password" name="pass"
               placeholder="Password"/>
    </li>
</ul>
 
<div class="topcoat-button-bar full" style="position: fixed; bottom: 0px;">
    <label class="topcoat-button-bar__item">
        <button class="topcoat-button--cta full" ng-click="doLogIn()">
            <span class="">Login</span>
        </button>
    </label>
</div>
```

As we can see in the application we're using a service to handle Http connections with the token information.

```javascript
'use strict';
 
G.factory('httpG', ['$http', '$window', function ($http, $window) {
    var serviceToken, serviceHost, tokenKey;
    tokenKey = 'token';
    if (localStorage.getItem(tokenKey)) {
        serviceToken = $window.localStorage.getItem(tokenKey);
    }
 
    $http.defaults.headers.post["Content-Type"] = "application/x-www-form-urlencoded";
 
    return {
        setHost: function (host) {
            serviceHost = host;
        },
 
        setToken: function (token) {
            serviceToken = token;
            $window.localStorage.setItem(tokenKey, token);
        },
 
        getToken: function () {
            return serviceToken;
        },
 
        removeToken: function() {
            serviceToken = undefined;
            $window.localStorage.removeItem(tokenKey);
        },
 
        get: function (uri, params) {
            params = params || {};
            params['_token'] = serviceToken;
            return $http.get(serviceHost + uri, {params: params});
        },
 
        post: function (uri, params) {
            params = params || {};
            params['_token'] = serviceToken;
 
            return $http.post(serviceHost + uri, params);
        }
    };
}]);
```

And that's all. You can see the full example in my [github](https://github.com/gonzalo123/token) account.
