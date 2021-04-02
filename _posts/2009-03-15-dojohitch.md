---
layout: post
permalink: /:year/:month/:day/:title/
title: "dojo.hitch"
date: "2009-03-15"
tags: 
  - "dojo"
  - "technology"
  - "web-development"
---

The first time I read about _dojo.hitch_ I didn't understand anything. I read in a book _dojo.hitch_ is a very important function and widely used into dojo library but I didn't understand why. I am using my learning of dojo to improve my skill in javascript. I remember the time when I hate javascript. I thought it was a very poor program language and everything I need to do with js was a nightmare. If browser was Netscape one way to to do one thing, if browser was IE other (of a other and a couple of hacks). But the times have changed. XHR and Ajax has reinvented the web. Web pages had became into Web Applications. Asynchronous request had opened our minds to a new generation of web applications, and all of it is thanks to javascript. Each day I spent learning js I see how much I was wrong.

Javascript literals are really powerful but the concept of scope is strange when you come from POO. _this.foo_ in js sounds like _$this->foo_ in PHP but its not the same.

For example in the following example:

```javascript
dojo.declare("gam.desktop", null, {
    variable1: 1,
    foo: function() {
        dojo.xhrPost({
            url: myUrl,
            load: function(responseObject, ioArgs){
                console.log(this.variable1);
            }),
            handleAs: "json"
        });
    });
```

console.log(this.variable1); will not display 1 in firebug console. Its strange for me when I started with dojo. The problem is that here this points to the scope of the function foo and not to the scope of class gam.desktop. You can do strange tricky-hacks to solve the problem with global variables or something like that, or use dojo.hitch.

```javascript
dojo.declare("gam.desktop", null, {
    variable1: 1,
    foo: function() {
        dojo.xhrPost({
            url: myUrl,
            load: dojo.hitch(this, function(responseObject, ioArgs){
                console.log(this.variable1);
            }),
            handleAs: "json"
        }));
    });
```

elegant isn't it?
