---
layout: post
title: "Live changes within node.js scripts without stop/start"
date: "2012-11-05"
tags: 
  - "javascript"
  - "node-js"
  - "technology"
---

Imagine that you are working within a nodejs project. This simple script:

```javascript
var CONF = ['item1', 'item2', 'item3'];
 
var last;
setInterval(function () {
    var next = CONF.indexOf(last) + 1;
    last = (CONF[next] == undefined) ? CONF[0] : CONF[next];
    console.log(last);
}, 1000);
```

If we run this script, we will see in the console one element of CONF each second. Simple, isn't it?. OK, imagine now we want to add one new element to the list (let's say item4). We can easily change the script, stop the execution and run again. OK but imagine that we cannot stop/start the script as many times as we want. What can we do?. We can store the CONF data into one external storage (Redis, for example), but today we are going to do something more easy. We are going to modify CONF in execution time. The idea is to open a TCP socket and let change CONF with a simple protocol.

If we change the script to: 

```javascript
var net = require('net');
var CONF_PORT = 9730;
var CONF = ['item1', 'item2', 'item3'];
 
var last;
setInterval(function () {
    var next = CONF.indexOf(last) + 1;
    last = (CONF[next] == undefined) ? CONF[0] : CONF[next];
    console.log(last);
}, 1000);
 
var serverConf = net.createServer(function (confSocket) {
    confSocket.on("data", function (data) {
        var dataAsString = data.toString().trim();
        switch (dataAsString.substr(0, 1)) {
            case '+':
                var userVariable = dataAsString.substr(1);
                if (CONF.indexOf(userVariable) < 0) {
                    CONF.push(userVariable);
                    confSocket.write("+ " + userVariable + " added\n");
                } else {
                    confSocket.write("+ " + userVariable + " already added\n");
                }
                break;
            case '-':
                var userVariable = dataAsString.substr(1);
                if (CONF.indexOf(userVariable) >= 0) {
                    CONF.splice(CONF.indexOf(userVariable), 1)
                    confSocket.write("- " + userVariable + " deleted\n");
                } else {
                    confSocket.write("- " + userVariable + " don't exists\n");
                }
                break;
            case '=':
                for (var i in CONF) {
                    confSocket.write(CONF[i] + "\n");
                }
                break;
        }
        confSocket.write("\n");
        confSocket.write("Number of elements: " + CONF.length + "\n");
        confSocket.end("\n");
    });
});
serverConf.listen(CONF_PORT);
```

You can see the script in action here:

[![youtube](https://img.youtube.com/vi/CWzfQvKTU8o\/0.jpg)](https://www.youtube.com/watch?v=CWzfQvKTU8o\)
