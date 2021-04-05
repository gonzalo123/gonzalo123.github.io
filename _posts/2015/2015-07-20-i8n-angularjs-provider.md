---
layout: post
title: "i18n AngularJS provider"
date: "2015-07-20"
tags: 
  - "angular"
  - "bower"
  - "i8n"
  - "ionic"
  - "js"
  - "angularjs"
  - "angularjs"
  - "technology"
---

There's more than one way to perform i18n translations within our AngularJS projects. IMHO the best one is https://angular-translate.github.io/, but today I'm going to show you how I'm doing translations in my small AngularJS projects (normally [Ionic](http://ionicframework.com/) projects).

I've packaged my custom solution and I also create one bower package ready to use via bower command line:

```commandline
bower install ng-i8n --save
```

First we add our provider

```html
<script src='lib/ng-i8n/dist/i8n.min.js'></script>
```

And now we add our new module ('gonzalo123.i18n') to our AngularJS project 

```javascript
angular.module('G', ['ionic', 'ngCordova', 'gonzalo123.i18n'])
```

Now we're ready to initialise our provider with the default language and translation data

```javascript
.config(function (i18nProvider, Conf) {
    i18nProvider.init(Conf.defaultLang, Conf.lang);
})
```

I like to use constants to store default lang and translation table, but it isn't necessary. We can just pass the default language and Lang object to our provider

```javascript
.constant('Conf', {
    defaultLang: 'es',
    lang: {
        HI: {
            en: 'Hello',
            es: 'Hola'
        }
    }
})
```

And that's all. We can translate key in templates (the project also provides a filter): 

```html
<h1 class="title">{{ 'HI' | i18n }}</h1>
```

And also inside our controllers \

```javascript
.controller('HomeController', function ($scope, i18n) {
    $scope.hi = i18n.traslate('HI');
})
```

If we need to change user language, we only need to trigger 'use' function: 

```javascript
.controller('HomeController', function ($scope, i18n) {
    $scope.changeLang = function(lang) {
        i18n.use(lang);
    };
})
```

Here we can see the code of our provider: 

```javascript
(function () {
    "use strict";
 
    angular.module('gonzalo123.i8n', [])
        .provider('i18n', function () {
            var myLang = {},
                userLang = 'en',
                translate;
 
            translate = function (key) {
                if (myLang.hasOwnProperty(key)) {
                    return myLang[key][userLang] || key;
                } else {
                    return key;
                }
            };
 
            this.$get = function () {
                return {
                    use: this.use,
                    translate: translate
                };
            };
 
            this.use = function (lang) {
                userLang = lang;
            };
 
            this.init = function (lang, conf) {
                userLang = lang;
                myLang = conf;
            };
        })
 
        .filter('i18n', ['i18n', function (i18n) {
            var i18nFilter = function (key) {
                return i18n.translate(key);
            };
 
            i8nFilter.$stateful = true;
 
            return i18nFilter;
        }])
    ;
})();
```

Anyway the project is in [my github account](https://github.com/gonzalo123/ngI8n)
