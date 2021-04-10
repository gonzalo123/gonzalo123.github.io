---
layout: post
title: "Web console with node.js"
date: "2011-05-16"
tags: 
  - "html5"
  - "node-js"
  - "php"
  - "technology"
  - "websockets"
  - "node"
  - "nodejs"
  - "websockets"
---

Continuing with my experiments of node.js, this time I want to create a Web console. The idea is simple. I want to send a few command to the server and I display the output inside the browser. I can do it entirely with PHP but I want to send the output to the browser as fast as they appear without waiting for the end of the command. OK we can do it [flushing](http://php.net/manual/es/function.flush.php) the output in the server but this solution normally crashes if we keep the application open for a long time. WebSockets again to the rescue. If we need a cross-browser implementation we need the [socket.io](http://socket.io/) library. Let’s start:

The node server is a simple websocket server. In this example we will launch each command with [spawn](http://nodejs.org/docs/v0.3.1/api/child_processes.html) function (require('child\_process').spawn) and send the output within the websoket. Simple and pretty straightforward.

```javascript
var sys   = require('sys'),
http  = require('http'),
url   = require('url'),
spawn = require('child_process').spawn,
ws    = require('./ws.js');
 
var availableComands = ['ls', 'ps', 'uptime', 'tail', 'cat'];
ws.createServer(function(websocket) {
    websocket.on('connect', function(resource) {
        var parsedUrl = url.parse(resource, true);
        var rawCmd = parsedUrl.query.cmd;
        var cmd = rawCmd.split(" ");
        if (cmd[0] == 'help') {
            websocket.write("Available comands: \n");
            for (i=0;i<availableComands.length;i++) {
                websocket.write(availableComands[i]);
                if (i< availableComands.length - 1) {
                    websocket.write(", ");
                }
            }
            websocket.write("\n");
 
            websocket.end();
        } else if (availableComands.indexOf(cmd[0]) >= 0) {
            if (cmd.length > 1) {
                options = cmd.slice(1);
            } else {
                options = [];
            }
             
            try {
                var process = spawn(cmd[0], options);
            } catch(err) {
                console.log(err);
                websocket.write("ERROR");
            }
 
            websocket.on('end', function() {
                process.kill();
            });
 
            process.stdout.on('data', function(data) {
                websocket.write(data);
            });
 
            process.stdout.on('end', function() {
                websocket.end();
            });
        } else {
             websocket.write("Comand not available. Type help for available comands\n");
             websocket.end();
        }
    });
   
}).listen(8880, '127.0.0.1');
```

The web client is similar than the example of my previous [post](http://gonzalo123.wordpress.com/2011/05/09/real-time-monitoring-php-applications-with-websockets-and-node-js/)

```javascript
var timeout = 5000;
var wsServer = '127.0.0.1:8880';
 
var ws;
 
 
function cleanString(string) {
    return string.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
}
 
 
function pad(n) {
    return ("0" + n).slice(-2);
}
 
var cmdHistory = [];
function send(msg) {
    if (msg == 'clear') {
        $('#log').html('');
        return;
    }
    try {
        ws = new WebSocket('ws://' + wsServer + '?cmd=' + msg);
        $('#toolbar').css('background', '#933');
        $('#socketStatus').html("working ... [<a href='#' onClick='quit()'>X</a>]");
        cmdHistory.push(msg);
        $('#log').append("<div class='cmd'>" + msg + "</div>");
        console.log("startWs:");
    } catch (err) {
        console.log(err);
        setTimeout(startWs, timeout);
    }
 
    ws.onmessage = function(event) {
        $('#log').append(util.toStaticHTML(event.data));
        window.scrollBy(0, 100000000000000000);
    };
 
    ws.onclose = function(){
        //console.log("ws.onclose");
        $('#toolbar').css('background', '#65A33F');
        $('#socketStatus').html('Type your comand:');
    }
}
 
function quit() {
    ws.close();
    window.scrollBy(0, 100000000000000000);
}
util = {
  urlRE: /https?:\/\/([-\w\.]+)+(:\d+)?(\/([^\s]*(\?\S+)?)?)?/g, 
 
  //  html sanitizer 
  toStaticHTML: function(inputHtml) {
    inputHtml = inputHtml.toString();
    return inputHtml.replace(/&/g, "&amp;")
                    .replace(/</g, "&lt;")
                    .replace("/n", "<br/>")
                    .replace(/>/g, "&gt;");
  }, 
 
  //pads n with zeros on the left,
  //digits is minimum length of output
  //zeroPad(3, 5); returns "005"
  //zeroPad(2, 500); returns "500"
  zeroPad: function (digits, n) {
    n = n.toString();
    while (n.length < digits) 
      n = '0' + n;
    return n;
  },
 
  //it is almost 8 o'clock PM here
  //timeString(new Date); returns "19:49"
  timeString: function (date) {
    var minutes = date.getMinutes().toString();
    var hours = date.getHours().toString();
    return this.zeroPad(2, hours) + ":" + this.zeroPad(2, minutes);
  },
 
  //does the argument only contain whitespace?
  isBlank: function(text) {
    var blank = /^\s*$/;
    return (text.match(blank) !== null);
  }
};
$(document).ready(function() {
  //submit new messages when the user hits enter if the message isnt blank
  $("#entry").keypress(function (e) {
    console.log(e);
    if (e.keyCode != 13 /* Return */) return;
    var msg = $("#entry").attr("value").replace("\n", "");
    if (!util.isBlank(msg)) send(msg);
    $("#entry").attr("value", ""); // clear the entry field.
  });
});
```

[![youtube](https://img.youtube.com/vi/Q5AcZj0Mml4\/0.jpg)](https://www.youtube.com/watch?v=Q5AcZj0Mml4\)

And that’s all. In fact we don’t need any line of PHP to perform this web console. Last year I tried to do something similar with PHP but it was a big mess. With node those kind of jobs are trivial. I don’t know if node.js is the future or is just another hype, but it’s easy. And cool. Really cool.

You can see the full code at Github [here](https://github.com/gonzalo123/webConsole). Anyway you must take care if you run this application on your host. You are letting user to execute raw unix commands. A bit of security layer would be necessary.
