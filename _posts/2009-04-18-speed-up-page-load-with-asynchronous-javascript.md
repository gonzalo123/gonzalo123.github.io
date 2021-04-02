---
layout: post
permalink: /:year/:month/:day/:title/
title: "Speed up page load with asynchronous javascript"
date: "2009-04-18"
tags: 
  - "technology"
  - "web-development"
---

JavaScript an important part of or web applications. Normally or web is not usable until js is full loaded. Almost all js framework implements an event to notice when the js is ready (dojo.addOnLoad(); $(function() {});) and the page is usable. The page is usable when js is loaded. Sometimes if you are working with dojo you need all js and maybe is better to create an splash screen until all is loaded. But sometimes your app don't need all js to be usable. I give you a example: I've been working in a project with jQuery. I load jQuery library from google cdn

```html
<script type="text/javascript" src="http://www.google.com/jsapi?key=mygoogleApiKey"></script>
<script type="text/javascript">
    google.load("jquery", "1.3.2");
</script>
```

jQuery is mandatory. The site doesn't work without this library.

```javascript
google.setOnLoadCallback(function() {
    $(function() {
        // application code ...
    });
});
```

I want to put rss feeds of some blogs on the left bar of the application and I need to do it with js. I'm going to use google's feed api to do it so:

```html
<script type="text/javascript" src="http://www.google.com/jsapi?key=myGoogleApiKey"></script>
<script type="text/javascript">
    google.load("jquery", "1.3.2");
    google.load("feeds", "1"); // <-------------
</script>
```

And I call to the js function to populates RSS feed into google's onready callback (setOnLoadCallback)

```javascript
google.setOnLoadCallback(function() { initFeeds() // <------------- $(function() { // application code ... }); });
```
This version of code works but it has a problem. All application will not be usable until initFeeds ends. The fedds on the left bar are cool but the user don't need them to use the application. It's just a decoration so: why we force user to wait until feeds are loaded? The solution is quite simple. Put a timer to free the main js file and populate feed asynchronously

```javascript
google.setOnLoadCallback(function() { setTimeout(initFeeds, 1000); <------------- 
    $(function() { // application code ... }); });
```
