---
layout: post
layout: post
title: "Handling private states within AngularJS applications"
date: "2015-03-19"
tags:
  - "technology"
  - "angularjs"
  - "ionic"
  - "javascript"
  - "js"
---

One typical task when we work with AngularJs application is login, and private states. We can create different states in our application. Something like this:

```javascript
.config(function ($stateProvider, $urlRouterProvider) {
    $stateProvider
        .state('state1', {
            url: '/state1',
            templateUrl: templates/state1.html,
            controller: 'State1Controller'
        })
        .state('state2', {
            url: '/state2',
            templateUrl: templates/state2.html,
            controller: 'State2Controller'
        })
    $urlRouterProvider.otherwise('/state1');
})
```

One way to create private states is using $stateChangeStart event. We can mark our private states with state parameters:

```javascript
.state('privateState1', {
        url: '/privateState1',
        templateUrl: templates/privateState1.html,
        controller: 'PrivateState1Controller',
        data: {
            isPublic: false
        }
    })
```

And then we can check out this parameters within $stateChangeStart event, doing one thing or another depending on token is present or not

```javascript
.run(function ($rootScope) {
    $rootScope.$on("$stateChangeStart", function (event, toState) {
        if (toState.data && toState.data.isPublic) {
            // do something here with localstorage and auth token
        }
    });
})
```

This method works, but last days, reading one project of Aaron K Saunders at [github](https://github.com/aaronksaunders/dcww), I just realised that there's another method. We can listen to $stateChangeError. Let me show you how can we do it.

The idea is to use [resolve](https://github.com/angular-ui/ui-router/wiki#resolve) in our private states. With resolve we can inject objects to our state's controllers, for example user information. This method is triggered before call to the controller, so that's a good place to check if token is present. If it isn't, then we can raise an error. This error will trigger $stateChangeError event, and here we can redirect the user to login state.

It sounds good, but we need to write resolve parameter in every private states, and that's bored. Especially when all states are private except login state. To by-pass this problem we can use abstract states. The idea is simple, we define one abstract state with "resolve" and then we create our private states under this abstract state.

Here we can see one example: login state isn't private, but state1 and state2 are private, indeed.

```javascript
.config(function ($stateProvider, $urlRouterProvider) {
    .state('login', {
        url: '/login',
        templateUrl: 'templates/login.html',
        controller: 'LoginController'
    })
    .state('private', {
        url: "/private",
        abstract: true,
        template: '<ui-view/>',
        resolve: {
            user: function (UserService) {
                return UserService.init();
            }
        }
    })
    .state('private.state1', {
        url: '/state1',
        templateUrl: 'templates/state1.html',
        controller: 'State1Controller'
    })
 
    .state('private.state2', {
        url: '/privateState2',
        templateUrl: 'templates/state2.html',
        controller: 'State2Controller'
    });
 
    $urlRouterProvider.otherwise('/private/privateState1');
})
```

Our UserService is a AngularJS service. This service provides three methods: init (the method that raises an error if token isn't present), login (to perform login and validate credentials), and logout (to remove token from localstorage and redirects to login state)

```javascript
.service('UserService', function ($q, $state) {
    var user = undefined;
 
    var UserService = {
        init: function () {
            var deferred = $q.defer();
 
            // do something here to get user from localstorage
 
            setTimeout(function () {
                if (user) {
                    deferred.resolve(user);
                } else {
                    deferred.reject({error: "noUser"});
                }
            }, 100);
 
            return deferred.promise;
        },
 
        login: function (userName, password) {
            // validate user and password here
        },
 
        logout: function () {
            // remove token from localstorage
            user = undefined;
            $state.go('login', {});
        }
    };
 
    return UserService
})
```

And finally the magic in $stateChangeError

```javascript
.run(function ($rootScope, $state) {
    $rootScope.$on('$stateChangeError',
        function (event, toState, toParams, fromState, fromParams, error) {
            if (error && error.error === "noUser") {
                $state.go('login', {});
            }
        });
})
```

And that's all. IMHO this solution is cleaner than $stateChangeStart method. What do you think?

**WARNING!** 
Before publishing this post I realize that this technique doesn't work 100% correctly. Maybe is my implementation but I tried to use it with an [ionic](http://ionicframework.com/) application and it doesn't work with android. Something kinda weird. It works with web applications, it works with IOS, but it doesn't work with Android. It looks like a bug (not sure about it). Blank screen instead of showing the template (but controller is loaded). We can see this anomalous situation using "ionic serve -l" (IOS ok and Android Not Ok)

To bypass this problem I tried a workaround. instead of using abstract states I create normal states, but to avoid to write again and again the resolve function to mark private states, I create a privateState provider

```javascript
.provider('privateState', function () {
    this.$get = function () {
        return {};
    };
 
    this.get = function(obj) {
        return angular.extend({
            resolve: {
                user: function (UserService) {
                    return UserService.init();
                }
            }
        }, obj);
    }
})
```

Now I can easily create private states without writing 'resolve' function.

```javascript
.config(function ($stateProvider, $urlRouterProvider, privateStateProvider) {
    $urlRouterProvider.otherwise('/home');
 
    $stateProvider
        .state('home', privateStateProvider.get({
            url: '/home',
            templateUrl: 'templates/home.html',
            controller: 'HomeController'
        }))
    ;
})
```
