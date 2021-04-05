---
layout: post
permalink: /:year/:month/:day/:title/
title: "Dojo and custom widgets"
date: "2009-02-15"
categories: 
  - "dojo"
  - "web-development"
---

I want to extend Dojo library with some custom widgets. Dojo widgets are really cool. You can use them from JavaScript and from HTML with the Dojo markup.

From js:

```javascript
var dialog = new dijit.Dialog({
            title: title,
            href: href,
            preventCache: false,
            parseOnLoad: true
            });
```

From HTML:

```html
<button DojoType="dijit.form.Button" type="submit" onClick='Desktop.login()'>Login</button>
```

Using Dojo widgets from HTML gives you errors when you validate your HTML files because Dojo attributes are non HTML attributes. If you are strict with HTML you have a problem using Dojo but its really easy to develop rich application with Dojo. so it's your choice ;). But remember: validation tools are just that: validation tools

For my tests I use Dojo from [CDN](http://en.wikipedia.org/wiki/Content_Delivery_Network). Dojo library is very big so I don' t want to upload all the library to my hosting. My hosting (a free one) has a monthly traffic usage limit so I want to limit the trafic to my hosting as far as I can (poor man techniques). Dojo library is in some CDNs (AOL and google for example). You only need to point your scripts to CDN and use cross domain (xd) of Dojo library and you are using the CDN. Really simple.

But If you create a custom widget you can't upload your widget to the CD. If you paid for a CDN (akamai for example) you can do it but according my poor man's techniques I use a free CDN so I can't upload anything to the CDN.

For example when you use a _dijit.form.TextBox_ with Dojo library pointing to google's CDN you are using the following url: _http://ajax.googleapis.com/ajax/libs/Dojo/1.2.3/dijit/form/TextBox.xd.js_ But if you are going to use your wonderfull custom widget _gam.widget.coolwidget_ your browser will try to load: _http://ajax.googleapis.com/ajax/libs/Dojo/1.2.3/gam/widget/coolwidget.xd.js_ and you will get a flaming 404 file not found error.

My first solution was upload all Dojo library to my hosting. With this solution I will can develop custom widgets without any problem but this is not the Dojo way. You can use xd version of Dojo and custom library in your server without problems. You only need to say Dojo in the configuration where are your widgets

```javascript
djConfig = {
    parseOnLoad: false,
    modulePaths:{
        gam: './js/gamjs'
    },
    baseUrl: '/'
};
```

With this configuration Dojo will continue trying to get _dijit.form.TextBox_ from _http://ajax.googleapis.com/ajax/libs/Dojo/1.2.3/dijit/form/TextBox.xd_.js but _gam.widget.coolwidget_ from _/js/gamjs/widget/coolwidget.xd.js._

Really easy. isn't it?
