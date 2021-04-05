---
layout: post
title: "Using OpenUI5 table and Angularjs"
date: "2015-01-12"
tags: 
  - "angularjs"
  - "html5"
  - "php"
  - "silex"
  - "symfony2"
  - "technology"
  - "angular"
  - "html5"
  - "javascrip"
  - "js"
  - "openui5"
  - "sap"
---

Last days I've been playing with [OpenUI5](https://sap.github.io/openui5/index.html). OpenUI5 is a web toolkit that [SAP](http://www.sap.com/) people has released as an open source project. I've read several good reviews about this framework, and because of that I started to hack a little bit with it. OpenUI5 came with a very complete set of [controls](https://openui5.hana.ondemand.com/#content/Controls/index.html). In this small example I want to use the "table" control. It's just a datagrid. This days I playing a lot with Angular.js so I wanted to use together OpenUI5's table control and Angularjs.

I'm not quite sure if it's a good decision to use together both frameworks. In fact we don't need Angular.js to create web applications using OpenUI5. OpenUI5 uses internally jQuery, but I wanted to hack a little bit and create one Angularjs directive to enclose one OpenUI5 datagrid.

First of all, we create one index.html. It's just a boilerplate with angular + ui-router + ui-bootstrap. We also start our OpenUi5 environment with de default theme and including the table library

```html
<!doctype html>
<html ng-app="G">
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
 
    <link rel="stylesheet" href="assets/bootstrap/dist/css/bootstrap.min.css">
 
    <script src="assets/angular/angular.js"></script>
    <script src="assets/angular-ui-router/release/angular-ui-router.js"></script>
    <script src="assets/angular-bootstrap/ui-bootstrap-tpls.js"></script>
 
    <script id='sap-ui-bootstrap' type='text/javascript'
            src="https://openui5.hana.ondemand.com/resources/sap-ui-core.js"
            data-sap-ui-theme='sap_bluecrystal'
            data-sap-ui-libs='sap.ui.commons, sap.ui.table'></script>
 
    <script src="js/ngOpenUI5.js"></script>
 
    <script src="js/app.js"></script>
    <link rel="stylesheet" href="css/app.css">
</head>
<body class="ng-cloak">
 
<div class="container">
 
    <div class="starter-template">
        <div ui-view></div>
    </div>
</div>
 
<script src="assets/html5shiv/dist/html5shiv.js"></script>
<script src="assets/respond/dest/respond.src.js"></script>
 
</body>
</html>
```

Then we create a directive enclosing the OpenUI5 needed code within a Angular module

```javascript
(function () {
    'use strict';
 
    angular.module('ng.openui5', [])
        .directive('openui5Table', function () {
 
            function renderColumns(columns, oTable) {
                for (var i = 0; i <= columns.length; i++) {
                    oTable.addColumn(new sap.ui.table.Column(columns[i]));
                }
            }
 
            var link = function (scope, element) {
 
                var oData = scope.model.data,
                    oTable = new sap.ui.table.Table(scope.model.conf),
                    oModel = new sap.ui.model.json.JSONModel();
 
                oModel.setData({modelData: oData});
                renderColumns(scope.model.columns, oTable);
 
                oTable.setModel(oModel);
                oTable.bindRows("/modelData");
                oTable.sort(oTable.getColumns()[0]);
 
                oTable.placeAt(element);
 
                scope.$watch('model.data', function (data) {
                    if (data) {
                        oModel.setData({modelData: data});
                        oModel.refresh();
                    }
                }, true);
 
            };
 
            return {
                restrict: 'E',
                scope: {model: '=ngModel'},
                link: link
            };
        });
}());
```

And now we can create a simple Angular.js using the ng.openui5 module. In this application we configure the table and fetch the data from an externar API server

```javascript
(function () {
    'use strict';
 
    angular.module('G', ['ui.bootstrap', 'ui.router', 'ng.openui5'])
 
        .value('config', {
            apiUrl: '/api'
        })
 
        .config(function ($stateProvider, $urlRouterProvider) {
            $urlRouterProvider.otherwise("/");
            $stateProvider
                .state('home', {
                    url: "/",
                    templateUrl: "partials/home.html",
                    controller: 'HomeController'
                });
        })
 
        .controller('HomeController', function ($scope, $http, $log, config) {
            $scope.refresh = function () {
                $http.get(config.apiUrl + '/gridData').success(function (data) {
                    $scope.datagrid.data = data;
                });
            };
 
            $scope.datagrid = {
                conf: {
                    title: "Table example",
                    navigationMode: sap.ui.table.NavigationMode.Paginator
                },
                columns: [
                    {
                        label: new sap.ui.commons.Label({text: "Last Name"}),
                        template: new sap.ui.commons.TextView().bindProperty("text", "lastName"),
                        sortProperty: "lastName",
                        filterProperty: "lastName",
                        width: "200px"
                    }, {
                        label: new sap.ui.commons.Label({text: "First Name"}),
                        template: new sap.ui.commons.TextField().bindProperty("value", "name"),
                        sortProperty: "name",
                        filterProperty: "name",
                        width: "100px"
                    }, {
                        label: new sap.ui.commons.Label({text: "Checked"}),
                        template: new sap.ui.commons.CheckBox().bindProperty("checked", "checked"),
                        sortProperty: "checked",
                        filterProperty: "checked",
                        width: "75px",
                        hAlign: "Center"
                    }, {
                        label: new sap.ui.commons.Label({text: "Web Site"}),
                        template: new sap.ui.commons.Link().bindProperty("text", "linkText").bindProperty("href", "href"),
                        sortProperty: "linkText",
                        filterProperty: "linkText"
                    }, {
                        label: new sap.ui.commons.Label({text: "Image"}),
                        template: new sap.ui.commons.Image().bindProperty("src", "src"),
                        width: "75px",
                        hAlign: "Center"
                    }, {
                        label: new sap.ui.commons.Label({text: "Gender"}),
                        template: new sap.ui.commons.ComboBox({
                            items: [
                                new sap.ui.core.ListItem({text: "female"}),
                                new sap.ui.core.ListItem({text: "male"})
                            ]
                        }).bindProperty("value", "gender"),
                        sortProperty: "gender",
                        filterProperty: "gender"
                    }, {
                        label: new sap.ui.commons.Label({text: "Rating"}),
                        template: new sap.ui.commons.RatingIndicator().bindProperty("value", "rating"),
                        sortProperty: "rating",
                        filterProperty: "rating"
                    }
 
                ]
            };
        })
    ;
}());
```

The API server is a simple Silex server

```php
<?php
include __DIR__ . '/../../vendor/autoload.php';
use Silex\Application;
 
$app = new Application();
$app->get("/", function (Application $app) {
 
$app->get('gridData', function (Application $app) {
    return $app->json([
        ['lastName' => uniqid(), 'name' => "Al", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 4, 'src' => "images/person1.gif"],
        ['lastName' => "Friese", 'name' => "Andy", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 2, 'src' => "images/person1.gif"],
        ['lastName' => "Mann", 'name' => "Anita", 'checked' => false, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 3, 'src' => "images/person1.gif"],
        ['lastName' => "Schutt", 'name' => "Doris", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 4, 'src' => "images/person1.gif"],
        ['lastName' => "Open", 'name' => "Doris", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 2, 'src' => "images/person1.gif"],
        ['lastName' => "Dewit", 'name' => "Kenya", 'checked' => false, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 3, 'src' => "images/person1.gif"],
        ['lastName' => "Zar", 'name' => "Lou", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 1, 'src' => "images/person1.gif"],
        ['lastName' => "Burr", 'name' => "Tim", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 2, 'src' => "images/person1.gif"],
        ['lastName' => "Hughes", 'name' => "Tish", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 5, 'src' => "images/person1.gif"],
        ['lastName' => "Lester", 'name' => "Mo", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 3, 'src' => "images/person1.gif"],
        ['lastName' => "Case", 'name' => "Justin", 'checked' => false, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 3, 'src' => "images/person1.gif"],
        ['lastName' => "Time", 'name' => "Justin", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 4, 'src' => "images/person1.gif"],
        ['lastName' => "Barr", 'name' => "Gaye", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 2, 'src' => "images/person1.gif"],
        ['lastName' => "Poole", 'name' => "Gene", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 1, 'src' => "images/person1.gif"],
        ['lastName' => "Ander", 'name' => "Corey", 'checked' => false, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 5, 'src' => "images/person1.gif"],
        ['lastName' => "Early", 'name' => "Brighton", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 3, 'src' => "images/person1.gif"],
        ['lastName' => "Noring", 'name' => "Constance", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 4, 'src' => "images/person1.gif"],
        ['lastName' => "Haas", 'name' => "Jack", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 2, 'src' => "images/person1.gif"],
        ['lastName' => "Tress", 'name' => "Matt", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "male", 'rating' => 4, 'src' => "images/person1.gif"],
        ['lastName' => "Turner", 'name' => "Paige", 'checked' => true, 'linkText' => "www.sap.com", 'href' => "http://www.sap.com", 'gender' => "female", 'rating' => 3, 'src' => "images/person1.gif"]
    ]);
});
$app->run();
```

And basically that's all. You can see the whole project within my [github](https://github.com/gonzalo123/openui5Table) account.

![table](/assets/images/localhost_8080___.jpg)
