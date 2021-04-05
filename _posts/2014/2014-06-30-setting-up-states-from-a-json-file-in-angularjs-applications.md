---
layout: post
title: "Setting up states from a json file in angularjs applications"
date: "2014-06-30"
tags: 
  - "angularjs"
  - "technology"
  - "angular"
  - "js"
  - "routeprovider"
---

Imagine a this simple angularjs application using angular-ui-router:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Example</title>
    <script src="bower_components/angular/angular.js"></script>
    <script src="bower_components/angular-ui-router/release/angular-ui-router.js"></script>
    <script src="js/app.js"></script>
 
</head>
<body ng-app="App" ng-controller="MainController">
 
<div ui-view></div>
</body>
</html>
```

```javascript
angular.module('App', ['ui.router'])
 
    .config(function ($stateProvider, $urlRouterProvider, routerProvider) {
        $stateProvider
            .state('home', {
                url: '/home',
                templateUrl: 'templates/home.html'
            });
 
        $urlRouterProvider.otherwise('/home');
    })
 
    .controller('MainController', function ($scope, router) {
        $scope.reload = function() {
            router.setUpRoutes();
        };
    })
;
```

We've defined only one state called "home". If we need more states we just add more within config() function. In this post we're going to try to add more states from a json file instead of hardcode the states within the code.

Let's create our json file with the states definitions: 

```json
{
    "xxx": {
        "url": "/xxx",
        "templateUrl": "templates/xxx.html"
    },
 
    "yyy": {
        "url": "/yyy",
        "templateUrl": "templates/yyy.html"
    },
 
    "zzz": {
        "url": "/zzz",
        "templateUrl": "templates/zzz.html"
    }
}
```

Now our application looks like this:

```javascript
angular.module('App', ['ui.router', 'Routing'])
 
    .config(function ($stateProvider, $urlRouterProvider, routerProvider) {
        $stateProvider
            .state('home', {
                url: '/home',
                templateUrl: 'templates/home.html'
            });
 
        $urlRouterProvider.otherwise('/home');
 
        routerProvider.setCollectionUrl('js/routeCollection.json');
    })
 
    .controller('MainController', function ($scope, router) {
        $scope.reload = function() {
            router.setUpRoutes();
        };
    })
;
```

As we can see now we're using 'Routing'

```javascript
angular.module('Routing', ['ui.router'])
    .provider('router', function ($stateProvider) {
 
        var urlCollection;
 
        this.$get = function ($http, $state) {
            return {
                setUpRoutes: function () {
                    $http.get(urlCollection).success(function (collection) {
                        for (var routeName in collection) {
                            if (!$state.get(routeName)) {
                                $stateProvider.state(routeName, collection[routeName]);
                            }
                        }
                    });
                }
            }
        };
 
        this.setCollectionUrl = function (url) {
            urlCollection = url;
        }
    })
 
    .run(function (router) {
        router.setUpRoutes();
    });
```

'Routing' provides us a provider called 'router' that fetch the json file and build the states.

That's a proof of concept. There's a couple of problems (please tell me if you know how to solve them):

- As far as we're loading states from a http connection, angular application don't have all the states when it starts, so we need to create at least the first state with the "old style"
- We can reload states with the application running. We also can add new states, but we cannot modify the existing ones.

you can see the one example project within my [github](https://github.com/gonzalo123/ngRouteProvider) account.
