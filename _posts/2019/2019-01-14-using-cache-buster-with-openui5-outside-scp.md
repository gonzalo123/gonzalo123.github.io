---
layout: post
title: "Using cache buster with OpenUI5 outside SCP"
date: "2019-01-14"
categories: 
  - "javascript"
  - "openui5"
  - "sap"
tags: 
  - "scp"
  - "cache"
  - "grunt"
  - "javascript"
  - "openui5"
  - "sap"
  - "sap-cloud-platform"
---

When we work with SPAs and web applications we need to handle with the browser's cache. Sometimes we change our static files but the client's browser uses a cached version of the file instead of the new one. We can tell the user: Please empty your cache to use the new version. But most of the times the user don't know what we're speaking about, and we have a problem. There's a technique called cache buster used to bypass this issue. It consists on to change the name of the file (or adding an extra parameter), basically to ensure that the browser will send a different request to the server to prevent the browser from reusing the cached version of the file.

When we work with [sapui5](https://sapui5.hana.ondemand.com/) application over [SCP](https://cloudplatform.sap.com/index.html), we only need to use the cachebuster version of sap-ui-core

```html
<script id="sap-ui-bootstrap"
      src="https://sapui5.hana.ondemand.com/resources/sap-ui-cachebuster/sap-ui-core.js"
      data-sap-ui-libs="sap.m"
      data-sap-ui-theme="sap_belize"
      data-sap-ui-compatVersion="edge"
      data-sap-ui-appCacheBuster=""
      data-sap-ui-preload="async"
      data-sap-ui-resourceroots='{"app": ""}'
      data-sap-ui-frameOptions="trusted">
</script>
```

With this configuration, our framework will use a "cache buster friendly" version of our files and SCP will serve them properly.

For example, when our framework wants the /dist/Component.js file, the browser will request /dist/~1541685070813~/Component.js to the server. And the server will server the file /dist/Component.js. As I said before when we work with SCP, our standard build process automatically takes care about it. It creates a file called sap-ui-cachebuster-info.json where we can find all our files with one kind of hash that our build process changes each time our file is changed.

```javascript
{
  "Component-dbg.js": 1545316733136,
  "Component-preload.js": 1545316733226,
  "Component.js": 1541685070813,
  ...
}
```

It works like a charm but I not always use SCP. Sometimes I use [OpenUI5](https://openui5.org/) in one nginx server, for example. So cache buster "doesn't work". That's a problem because I need to handle with browser caches again each time we deploy the new version of the application. I wanted to solve the issue. Let me explain how I did it.

Since I was using one Lumen/PHP server to the backend, my first idea was to create a dynamic route in Lumen to handle cache-buster urls. With this approach I know I can solve the problem but there's something that I did not like: I'll use a dynamic server to serve static content. I don't have a huge traffic. I can use this approach but it isn't beautiful.

My second approach was: Ok I've got a sap-ui-cachebuster-info.json file where I can see all the files that cache buster will use and their hashes. So, Why not I create those files in my build script. With this approach I will create the full static structure each time I deploy de application, without needing any server side scripting language to generate dynamic content. OpenUI5 uses grunt so I can create a simple grunt task to create my files.

```javascript
'use strict';
 
var fs = require('fs');
var path = require('path');
var chalk = require('chalk');
 
module.exports = function(grunt) {
  var name = 'cacheBuster';
  var info = 'Generates Cache buster files';
 
  var cacheBuster = function() {
    var config = grunt.config.get(name);
    var data, t, src, dest, dir, prop;
 
    data = grunt.file.readJSON(config.src + '/sap-ui-cachebuster-info.json');
    for (prop in data) {
      if (data.hasOwnProperty(prop)) {
        t = data[prop];
        src = config.src + '/' + prop;
        dest = config.src + '/~' + t + '~/' + prop;
        grunt.verbose.writeln(
            name + ': ' + chalk.cyan(path.basename(src)) + ' to ' +
            chalk.cyan(dest) + '.');
        dir = path.dirname(dest);
        grunt.file.mkdir(dir);
        fs.copyFileSync(src, dest);
      }
    }
  };
 
  grunt.registerMultiTask(name, info, cacheBuster);
};
```

I deploy my grunt task to npm so when I need to use it I only need to:

Install the task

> npm install gonzalo123-cachebuster

Add the task to my gruntfile

```javascript
require('gonzalo123-cachebuster')(grunt);
```

and set up the path where ui5 task generates our dist files 

```javascript
grunt.config.merge({
  pkg: grunt.file.readJSON('package.json'),
  ...
  cacheBuster: {
    src: 'dist'
  }
});
```

And that's all. My users with enjoy (or suffer) the new versions of my applications without having problems with cached files.

Grunt task available in my [github](https://github.com/gonzalo123/cachebuster)
