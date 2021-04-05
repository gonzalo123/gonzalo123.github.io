---
layout: post
title: "i18n angular2 service"
date: "2016-08-01"
tags: 
  - "angular2"
  - "technology"
  - "i18n"
  - "ionic2"
  - "typescript"
---

This days I'm learning [angular2](https://angular.io/) and [ionic2](http://ionic.io/2). My first impression was bad. Too many new things: angular2, typescript, new project architecture, ... but after passing those bad moments I started to like angular2. One of the first things that I want to work with is internationalization. There're several i18n providers for angular2. For example [ng2-translate](https://github.com/ocombe/ng2-translate), but I want to build something similar to a angular i18n provider that I'd created [time ago](https://gonzalo123.com/2015/07/20/i8n-angularjs-provider/).

The idea is first configure the service

```typescript
import {Component} from "@angular/core";
import {Platform, ionicBootstrap} from "ionic-angular";
import {StatusBar} from "ionic-native";
import {HomePage} from "./pages/home/home";
import {I18nService} from "./services/i18n/I18nService";
import {lang} from "./conf/conf";
 
@Component({
    template: '<ion-nav [root]="rootPage"></ion-nav>'
})
export class MyApp {
    rootPage:any = HomePage;
 
    constructor(platform:Platform, i18n:I18nService) {
        platform.ready().then(() => {
            i18n.init(lang);
            StatusBar.styleDefault();
        });
    }
}
 
ionicBootstrap(MyApp, [I18nService]);
```

We can see that we're using one configuration from conf/conf.ts 

```typescript
var lang = {
    availableLangs: {
        en: 'English'
    },
    defaultLang: 'en',
    lang: {
        "true": {
            en: "True",
            es: "Verdadero"
        },
        "false": {
            en: "False",
            es: "Falso"
        },
    }
};
 
export {lang};
```

Now we can translate texts using: 

```typescript
import {Component} from "@angular/core";
import {I18nService} from "../../services/i18n/I18nService";
 
@Component({
    templateUrl: 'build/pages/home/home.html'
})
export class HomePage {
    constructor(private i18n:I18nService) {
    }
 
    label1:string = this.i18n.translate('true');
    label2:string = 'true';
     
    ngOnInit() {
        setInterval(() => {
            this.label2 = this.i18n.translate(this.label2 == 'false' ? 'true' : 'false');
        }, 2000);
    }
}
```

We also need a Pipe to use i18n within templates: 

```typescript
{{ "true" | i18n }}
```

This is the service: 
```typescript
import {Injectable, Pipe} from "@angular/core";
 
@Injectable()
export class I18nService {
 
    private conf:any;
    private userLang:string;
    private lang:any;
 
    setUserLang(lang:string):void {
        this.userLang = lang;
    }
 
    init(lang:any):void {
        this.conf = lang;
        this.setUserLang(lang.defaultLang);
    }
 
    translate(key:string):string {
        if (typeof this.conf.lang !== 'undefined' && this.conf.lang.hasOwnProperty(key)) {
            return this.conf.lang[key][this.userLang] || key;
        } else {
            return key;
        }
    }
}
 
@Pipe({
    name: 'i18n',
    pure: false
})
export class I18nPipe {
    constructor(private i18n:I18nService) {
    }
 
    transform(key:string) {
        return this.i18n.translate(key);
    }
}
```

The code is in my [github](https://github.com/gonzalo123/angular2-i18n) account. I also have tried to created a npm package and install it using

The code is in my github account. I also have tried to created a npm package and install it using

```commandline
npm install angular2-i18n --save
```

I don’t know why it doesn’t work when I use the npm installed package and I works like a charm when I use the code in my project directory

```typescript
import {I18nService} from "./services/i18n/I18nService"; // works
import {I18nService} from "angular2-i18n/src/I18nService"; // don't work
```

I also have problems with unit tests. It works when I run the tests locally (it works on my machine :) ) but it doesn't work when I run them in [travis](https://travis-ci.org/gonzalo123/angular2-i18n). Any help with those two problems would be appreciated.
