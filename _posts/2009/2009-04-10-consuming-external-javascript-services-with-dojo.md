---
layout: post
permalink: /:year/:month/:day/:title/
title: "Consuming external javascript services with Dojo"
date: "2009-04-10"
tags: 
  - "dojo"
  - "technology"
  - "web-development"
---

I want to include goggle's GSbookBar in one dokjo application. Unfortunately this API doesn't have a JSONP interface. I need to include one external js file and use GSbookBar object. It's really simple but I want to so something different. The book bar appears only in one pop up of one tab in my application. It isn't a main part so I don't want to load the external script every time the user loads the application.

Dojo has [dojo.io.script.get](http://dojocampus.org/explorer/#Dojo_IO_Script) to load external scripts. Its a cool way to do the traditional:

```javascript
var scriptElement = document.createElement("script");
scriptElement.setAttribute("id", "jsonScript");
scriptElement.setAttribute("src", "http://external/script.js");
scriptElement.setAttribute("type", "text/javascript");
document.documentElement.firstChild.appendChild(scriptElement);

```

dojo.io.script.get is an elegant way load external scripts. It provides us the same interface to consume JSONP and no-JSONP scripts. One important thing when we include some external js functionality in our projects is that we need to know when the external js script is loaded to use its functionality. To do it dojo.io.script.get fires load event. In scripts without JSONP interface dojo uses checkString. Dojo evaluates checkString "typeof(" + checkString + ") != 'undefined' and fires load event when is loaded.

The trick I use to load external scripts is find one js object in the script and use it as checkString. In this example the checkString is GSbookBar js object

```javascript
GSbookBar = undefined;
dojo.io.script.get({
    url:"http://www.google.com/uds/solutions/bookbar/gsbookbar.js?mode=new",
    checkString: "GSbookBar",
    load: dojo.hitch(this, function(){
    window._uds_bbw_donotrepair = true;
    var bookBar;
    var options = {
        largeResultSet : !true,
        horizontal : true,
        autoExecuteList : {
            cycleTime : GSbookBar.CYCLE_TIME_MEDIUM,
            cycleMode : GSbookBar.CYCLE_MODE_RANDOM,
            thumbnailSize: GSbookBar.thumbnailsSmall,
            executeList : [data.title, data.author]
        }
    }
    bookBar = new GSbookBar(this.bookBar, options);
    dialog.resize(); // hack because of a bug in dojo 1.3
    dialog.show();  // I have created dialog dijit widget previosly
})
```
