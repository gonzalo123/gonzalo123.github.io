---
layout: post
title: "Upgrading Cordova-iOS apps outside Apple Store"
date: "2014-10-13"
tags: 
  - "android"
  - "angularjs"
  - "cordova"
  - "ios"
  - "technology"
  - "apple"
  - "apple-store"
  - "ionic"
  - "ios"
  - "ios-applications"
---

In one of my last post I explained how to [upgrade Cordova-Android apps outside Google Play Store with angularjs](http://gonzalo123.com/2014/07/28/upgrading-android-apps-outside-google-play-store-with-angularjs/ "Upgrading Cordova-Android apps outside Google Play Store with angularjs"). Today is the turn of iOS applications.

If you work with in-house iOS applications you need to define a distribution strategy (you cannot use Apple Store, indeed). Apple provides documentation to do it. Basically we need to place our ipa file in addition to the plist file (generated when we archive our application with xCode). I'm not going to explain how to do it here. As I said before it's well documented. Here I'm going to explain how to do the same trick than the Android's post but now with our iOS application.

With iOS, to install the application, we only need to provide the iTunes link to our plist application (something like this: itms-services://?action=download-manifest&url=http://url.to.plist) and open it with the InAppBrowser plugin.

First we install the InAppBrowser plugin:

```commandline
$ cordova plugin add https://git-wip-us.apache.org/repos/asf/cordova-plugin-inappbrowser.git
```

And now we only need to open the url using the plugin:

```commandline
var iosPlistUrl = 'http://url.to.plist';
cordova.exec(null, null, "InAppBrowser", "open", [encodeURI("itms-services://?action=download-manifest&url=" + iosPlistUrl), "_system"]);
```

We can use exactly the same angularJs used the the previous post to check the version and the same server-side verification.

We also can detect the platform with Device plugin and do one thing or another depending on we are using Android or iOS.

[Here](https://github.com/gonzalo123/iOS-cordova-autoupgrade) you can see one example using ionic framework. This example uses one $http interceptor to send version number within each request and we trigger 'wrong.version' to the event dispatcher when it detects a wrong versions between client and server

```javascript
angular.module('G', ['ionic'])
 
    .value('appConf', {
        version: 1,
        apiHost: 'http://localhost:8080'
    })
 
    .config(function ($httpProvider, $urlRouterProvider, $stateProvider) {
        $httpProvider.interceptors.push('versionInterceptor');
 
        $stateProvider
            .state('home', {
                url: '/home',
                templateUrl: 'partials/home.html',
                controller: 'HomeController'
            })
            .state('upgrade', {
                url: '/upgrade',
                templateUrl: 'partials/upgrade.html',
                controller: 'UpgradeController'
            })
        ;
 
        $urlRouterProvider.otherwise('/home');
 
    })
 
    .run(function ($ionicPlatform, $rootScope, $state) {
        $ionicPlatform.ready(function () {
            if (window.cordova && window.cordova.plugins.Keyboard) {
                cordova.plugins.Keyboard.hideKeyboardAccessoryBar(true);
            }
            if (window.StatusBar) {
                StatusBar.styleDefault();
            }
        });
 
        $rootScope.$on('wrong.version', function () {
            $state.go("upgrade");
        });
    })
 
    .controller('HomeController', function ($scope, $http, appConf) {
        $scope.someAction = function () {
            $http.get(appConf.apiHost + "/hello", function (data) {
                alert(data);
            });
        }
    })
 
    .controller('UpgradeController', function ($scope) {
        $scope.upgrade = function () {
            cordova.exec(null, null, "InAppBrowser", "open", [encodeURI("itms-services://?action=download-manifest&url=https://path/to/plist.plist"), "_system"]);
        }
    })
 
    .factory('versionInterceptor', function ($rootScope, appConf) {
        var versionInterceptor = {
            request: function (config) {
                config.url = config.url + '?_version=' + appConf.version;
 
                return config;
            },
            responseError: function(response) {
                if (response.status == 410) {
                    $rootScope.$emit('wrong.version');
                }
            }
        };
 
        return versionInterceptor;
    })
;
```
