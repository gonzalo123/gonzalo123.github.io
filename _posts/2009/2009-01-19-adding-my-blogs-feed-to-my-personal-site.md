---
layout: post
permalink: /:year/:month/:day/:title/
title: "Adding my blog's feed to my personal site"
date: "2009-01-19"
tags: 
  - "web-development"
---

I have my [personal](http://gonzalo123.co.cc/ "personal") web page hosted in a free hosting service. Basically it's a [dojo](http://dojotoolkit.org/ "dojo") application working with a [Zend Framework](http://framework.zend.com/ "Zend Framework") Backend. I had the idea of include a the last posts of my blog from the rss to the front page (original isn't it?), and I start to develop it.

I started using Zend\Feed to fetch the RSS from my blog. The idea of my [personal](http://gonzalo123.co.cc/ "personal") site was to develop a web application based on [dojo](http://dojotoolkit.org/ "dojo") framework so I decide to use the dojo way using `dojox.data.AtomReadStore.`That sounds cool but I faced with the first problem. Due to security reasons XHR requests can't be done to a different domain. My [personal](http://gonzalo123.co.cc/ "personal") page and the url of the blog's feed were under different domain.

## Problem 1: XHR with different domains.

To solve this problem I must create a proxy with PHP to RSS feed in my [personal](http://gonzalo123.co.cc/ "personal") page. With this proxy XHR requests will be in the same domain, so `dojox.data.AtomReadStore` will work correctly. the proxy worked perfectly in my local computer but when I upload it to my hosting I realized that it didn't work.

## Problem 2: The hosting of my project doesn't allow php's socket functions

As I see in php.ini. some PHP functions (socket functions for example) are disallowed in the server. Because of that my code does not work. Surfing into Zend framework source code (open source is cool). I saw Zend\_Feed uses Zend\_Http\_Client\_Adapter\_Socket. My hosting allows me to use curl extension instead of socket functions. But Zend Framework does not have a Zend\_Http\_Client\_Adapter\_Curl. So other problem.

## Problem 3: Zend\_Http\_Client with Curl

As I read in some newsgroup my problem with my hosting, curl and sockets was not only mine. Zend Framework doesn't not have a curl adapter in the production tree but they have a Zend\_Http\_Client\_Adapter\_Curl in the incubator. This adapter is not fully tested and it can crash behind proxies and things like that but it sounds good for my situation. Using this adapter the code works in my local server and in my hosting. that's sounds good but there is other problem

## Problem 4: CPU usage

Looking in the statistics of my site in the hosting I realized that the CPU usage was hight. The application is a client side application and this CPU usage graph shows me the problem. Curl functions are slow and my rss proxy is consuming too much cpu time. It sounds I am under a no-solution problem. I need a proxy to avoid XHR cross domain restriction but this proxy uses too much CPU in my poor-man- hosting. Solution: turn the logic to the client and and do all the job with javascript.

Finding some RSS reader with js I faced with [GFdynamicFeedControl](http://www.google.com/uds/solutions/dynamicfeed/ "GFdynamicFeedControl"). Really easy to use. It also has a wizard to generate code to copy and paste. Of course the generated code didn't work in my application but hacking a bit I have finally my RSS feeds in my [personal](http://gonzalo123.co.cc/ "personal") page. [GFdynamicFeedControl](http://www.google.com/uds/solutions/dynamicfeed/ "GFdynamicFeedControl") allow you to mix different RSS so y also include the rss of my [github](http://github.com/gonzalo123/gam/tree/master "github") account with the source code.

```html
<div id="feed-control">
    <span style="color:#676767;font-size:11px;margin:10px;padding:4px;">Loading...</span></div>
@import url("http://www.google.com/uds/solutions/dynamicfeed/gfdynamicfeedcontrol.css");
 
google.load('feeds', '1');
dojo.addOnLoad(function(){
    var options = {
        stacked : true,
        vertical : true,
        title : "Feeds"
    }
    new GFdynamicFeedControl(gamJs.feeds, 'feed-control', options);
})
```
