---
layout: post
permalink: /:year/:month/:day/:title/
title: "JSONP. JSON with Padding"
date: "2009-01-20"
categories: 
  - "web-development"
---

[JSON](http://en.wikipedia.org/wiki/JSON "JSON") is a really good solution to send data from server to client in web applications. Almost every program language has his own json encode and decode. PHP allows natively to do [this](http://es2.php.net/json "this") encoding from php arrays and objets to JSON. And Zend Framework implements [Zend\_JSON](http://framework.zend.com/manual/en/zend.json.html "Zend_JSON") who also allows you to encode/decode from XML.

When I was developing a rss client to my personal home page. I discover JSONP services. Those services are really easy to use and really easy to integrate them into your projects:

```
If the following url gives you a json data url: www.server.com/service {"identifier":"id","items":\[{"id":"18","title":" Asterisk. The future of telephony","author":"Jim Van Meggelen","bookyear":"2008","why":""}\]}
```
It is very helpfully when

```
url: www.server.com/service?callback=myfunction myfunction({"identifier":"id","items":\[{"id":"18","title":" Asterisk. The future of telephony","author":"Jim Van Meggelen","bookyear":"2008","why":""}\]})
```

with this if you develop the function 'myfunction' with a few javascrip lines you can consume those jsonp webservices.

This is the source code I use to read

```javascript
function googleSearch(bibkeys) {
  var scriptElement = document.createElement(”script”);
  scriptElement.setAttribute(”id”, “jsonScript”);
  scriptElement.setAttribute(”src”, “http://books.google.com/books?bibkeys=” + 
    bibkeys + “&jscmd=viewapi&callback=googleCallback”);
  scriptElement.setAttribute(”type”, “text/javascript”);
  document.documentElement.firstChild.appendChild(scriptElement);
}
function googleCallback(booksInfo) { // do what you want with the booksInfo }
```
