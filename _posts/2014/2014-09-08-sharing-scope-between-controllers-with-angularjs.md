---
layout: post
title: "Sharing $scope between controllers with AngularJs"
date: "2014-09-08"
tags: 
  - "angularjs"
  - "technology"
  - "js"
---

Angular creates one [$scope](https://docs.angularjs.org/guide/scope) object for each controller. We also have a $rootScope accesible from every controllers. But, can we access to one controller's $scope from another controller? The sort answer is no. Also if our application needs to access to another controller's $scope, we probably are doing something wrong and we need to re-think our problem. But anyway it's possible to access to another controller's $scope if we store it within a service. Let me show you and example.

Imagine this little example:

```html
<!doctype html>
<html ng-app="app">
<head>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.0-beta.18/angular.js"></script>
    <script src="app.js"></script>
</head>
<body>
 
<div ng-controller="OneController">
    <h2>OneController</h2>
    <button ng-click="buttonClick()">
        buttonClick on current scope
    </button>
</div>
 
<div ng-controller="TwoController">
    <h2>TwoController</h2>
    <button ng-click="buttonClick()">
        buttonClick on current scope
    </button>
</div>
</body>
</html>
```

As we can see we define two controllers: "OneController" and "TwoController".

That's the application:

```javascript
var app = angular.module('app', []);
 
app.controller('OneController', function ($scope) {
    $scope.variable1 = "One";
 
    $scope.buttonClick = function () {
        console.log("OneController");
        console.log("$scope::variable1", $scope.variable1);
    };
});
 
app.controller('TwoController', function ($scope) {
    $scope.variable1 = "Two";
 
    $scope.buttonClick = function () {
        console.log("TwoController");
        console.log("$scope::variable1", $scope.variable1);
    };
});
```

If we need to access to another controller’s $scope we need to store those scopes within a service. For example with this minimal service:

```javascript
app.factory('Scopes', function ($rootScope) {
    var mem = {};
 
    return {
        store: function (key, value) {
            mem[key] = value;
        },
        get: function (key) {
            return mem[key];
        }
    };
});
```

And now we need to store the $scope in the service: 

```javascript
app.controller('OneController', function ($scope, Scopes) {
    Scopes.store('OneController', $scope);
    ...
});
app.controller('TwoController', function ($scope, Scopes) {
    Scopes.store('TwoController', $scope);
    ...
});
```

And now we can access to another's $scope

Here the full example:

```html
<!doctype html>
<html ng-app="app">
<head>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.0-beta.18/angular.js"></script>
    <script src="app.js"></script>
</head>
<body>
 
<div ng-controller="OneController">
    <h2>OneController</h2>
    <button ng-click="buttonClick()">
        buttonClick on current scope
    </button>
    <button ng-click="buttonClickOnTwoController()">
        buttonClick on TwoController's scope
    </button>
</div>
 
<div ng-controller="TwoController">
    <h2>TwoController</h2>
    <button ng-click="buttonClick()">
        buttonClick on current scope
    </button>
    <button ng-click="buttonClickOnOneController()">
        buttonClick on OneController's scope
    </button>
</div>
</body>
</html>
```

and app.js 

```javascript
var app = angular.module('app', []);
 
app.run(function ($rootScope) {
    $rootScope.$on('scope.stored', function (event, data) {
        console.log("scope.stored", data);
    });
});
app.controller('OneController', function ($scope, Scopes) {
 
    Scopes.store('OneController', $scope);
 
    $scope.variable1 = "One";
 
    $scope.buttonClick = function () {
        console.log("OneController");
        console.log("OneController::variable1", Scopes.get('OneController').variable1);
        console.log("TwoController::variable1", Scopes.get('TwoController').variable1);
        console.log("$scope::variable1", $scope.variable1);
    };
 
    $scope.buttonClickOnTwoController = function () {
        Scopes.get('TwoController').buttonClick();
    };
});
app.controller('TwoController', function ($scope, Scopes) {
 
    Scopes.store('TwoController', $scope);
 
    $scope.variable1 = "Two";
 
    $scope.buttonClick = function () {
        console.log("TwoController");
        console.log("OneController::variable1", Scopes.get('OneController').variable1);
        console.log("TwoController::variable1", Scopes.get('TwoController').variable1);
        console.log("$scope::variable1", $scope.variable1);
    };
 
    $scope.buttonClickOnOneController = function () {
        Scopes.get('OneController').buttonClick();
    };
});
app.factory('Scopes', function ($rootScope) {
    var mem = {};
 
    return {
        store: function (key, value) {
            $rootScope.$emit('scope.stored', key);
            mem[key] = value;
        },
        get: function (key) {
            return mem[key];
        }
    };
});
```

You can also see it running [here](http://jsfiddle.net/gonzalo123/myc5rcLw/)
