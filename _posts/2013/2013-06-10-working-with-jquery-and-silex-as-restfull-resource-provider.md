---
layout: post
title: "Working with jQuery and Silex as RestFull Resource provider"
date: "2013-06-10"
tags: 
  - "jquery"
  - "js"
  - "php"
  - "symfony"
  - "technology"
  - "silex"
  - "technology"
---

The previous [post](http://gonzalo123.com/2013/06/03/working-with-angularjs-and-silex-as-resource-provider/) was about how to use AngularJS resources with Silex. AngularJS is great and when I need to switch back to jQuery it looks like I go back 10 years in web development, but business is business and I need to live with jQuery too. Because of that this post is about how to use the Silex RestFull resources from the previous post, now with jQuery. Let's start:

We're going to write a simple javascript object to handle the RestFull resource using jQuery:

```javascript
var Resource = (function (jQuery) {
    function Resource(url) {
        this.url = url;
    }
 
    Resource.prototype.query = function (parameters) {
        return jQuery.getJSON(this.url, parameters || {});
    };
 
    Resource.prototype.get = function (id, parameters) {
        return jQuery.getJSON(this.url + '/' + id, parameters || {});
    };
 
    Resource.prototype.remove = function (id, parameters) {
        return jQuery.ajax({
            url:this.url + '/' + id,
            xhrFields:JSON.stringify(parameters || {}),
            type:'DELETE',
            dataType:'json'
        });
    };
 
    Resource.prototype.update = function (id, parameters) {
        return jQuery.post(this.url + '/' + id, JSON.stringify(parameters || {}), 'json');
    };
 
    Resource.prototype.create = function (parameters) {
        return jQuery.post(this.url, JSON.stringify(parameters || {}), 'json');
    };
 
    return Resource;
})(jQuery);
```

As you can see the library returns jQuery ajax promises, so we can use done() and error() callbacks to work with the server's data.

Here our application example:

```javascript
var host = document.location.protocol + '//' + document.location.host;
var resource = new Resource(host + '/api/message/resource');
 
resource.create({ id: 10, author: 'myname', message: 'hola'}).done(function (data) {
    console.log("create element", data);
});
 
resource.query().done(function (data) {
   console.log("query all", data);
});
 
resource.update(10, {message: 'hi'}).done(function (data) {
    console.log("update element 1", data);
});
 
resource.get(10).done(function (data) {
    console.log("get element 1", data);
});
 
resource.remove(10).done(function (data) {
    console.log("remove element 1", data);
});
 
resource.query().done(function (data) {
    console.log("query all", data);
});
```

And that's all. You can get the full code of the example from my [github](https://github.com/gonzalo123/jQueryRestResource) account
