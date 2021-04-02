---
layout: post
permalink: /:year/:month/:day/:title/
title: "mixin dojo and google api libraries"
date: "2009-04-18"
categories: 
  - "dojo"
  - "technology"
  - "web-development"
---

I'm working in a dojo application and I also need to use some goolge api like feeds, maps and books. Dojo is very versatile adding new components using dojo.require. when all dojo components are loaded dojo.addOnLoad callback is fired.

```javascript
dojo.require("dojo.io.script");
dojo.require("dojo.data.ItemFileReadStore");
dojo.addOnLoad(function(){
    console.log('all is ready');
});
```

dojo.require is great. It also alloy us to create nested onLoad callbacks

```javascript
dojo.require("dojo.io.script");
dojo.require("dojo.data.ItemFileReadStore");

dojo.addOnLoad(function(){  // &lt;------------
    console.log('all is ready1');
    dojo.require("dijit.TitlePane");
    dojo.require("dijit.Dialog");

    dojo.addOnLoad(function(){ // &lt;------------
        console.log('all is ready 2');
     });
});
```

google API uses his own loader google.load.

```javascript
google.load("jquery", "1.3.2");
google.load("feeds", "1");
google.setOnLoadCallback(function() {
    console.log('all is ready');
});
```

If your application uses both libraries you must take care about onLoad callback of dojo and google because they are not the same. You can create your own dojo widget for loading google api and forget google.setOnLoadCallback but I want to use libraries as standard as I can (without hacking code every time I have a problem). So I prefer to live with both callbacks. You can try to do something like this:

```javascript
google.load("dojo", "1.3");
google.load("feeds", "1");

dojo.require("dijit.layout.BorderContainer");
dojo.require("dijit.layout.TabContainer");
dojo.require("dijit.layout.ContentPane");

dojo.addOnLoad(function(){
    // start dojo application
});
```

This code will work but sometimes will crash. Then you press F5 and all works. As far as I know you can't do nested onLoad callbacks with google (it's a pity), so first you must load google api and load dojo components when google api is ready So it's better to use this other version of code:

```javascript
google.load("dojo", "1.3");
google.load("feeds", "1");
google.setOnLoadCallback(function() {
    dojo.require("dijit.layout.BorderContainer");
    dojo.require("dijit.layout.TabContainer");
    dojo.require("dijit.layout.ContentPane");

    dojo.addOnLoad(function(){
        // start dojo application
    });

});
```
