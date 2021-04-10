---
layout: post
title: "Real time notifications with PHP"
date: "2011-03-14"
tags: 
  - "jquery"
  - "php"
  - "comet"
  - "jquery"
  - "js"
---

Real time communications are cool, isn’t it? Something impossible to do five years ago now (or almost impossible) is already available. Nowadays we have two possible solutions. WebSockets and Comet. WebSockets are probably the best solution but they've got two mayor problems:

- Not all browsers support them.
- Not all proxy servers allows the communications with websokets.

Because of that I prefer to use comet (at least now). It’s not as good as websockets but pretty straightforward ant it works (even on IE). Now I’m going to explain a little script that I’ve got to perform a comet communications, made with PHP. Probably it’s not a good idea to use it in a high traffic site, but it works like a charm in a small intranet. If you want to use comet in a high traffic site maybe you need have a look to [Tornado](http://www.tornadoweb.org/), [twisted](http://twistedmatrix.com/trac/), [node.js](http://nodejs.org/) or other comet dedicated servers.

Normally when we are speaking about real-time communications, all the people are thinking about a chat application. I want to build a simpler application. A want to detect when someone clicks on a link. Because of that I will need a combination of HTML, PHP and JavaScript. Let’s start:

For the example I’ll use jquery library, so we need to include the library in our HTML file. It will be a blend of JavaScrip and PHP:

```html
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>Comet Test</title>
    </head>
    <body>
        <p><a class='customAlert' href="#">publish customAlert</a></p>
        <p><a class='customAlert2' href="#">publish customAlert2</a></p>
        <script src="http://ajax.googleapis.com/ajax/libs/jquery/1.5/jquery.min.js" type="text/javascript"></script>
        <script src="NovComet.js" type="text/javascript"></script>
        <script type="text/javascript">
NovComet.subscribe('customAlert', function(data){
    console.log('customAlert');
    //console.log(data);
}).subscribe('customAlert2', function(data){
    console.log('customAlert2');
    //console.log(data);
});
 
$(document).ready(function() {
    $("a.customAlert").click(function(event) {
        NovComet.publish('customAlert');
    });
     
    $("a.customAlert2").click(function(event) {
        NovComet.publish('customAlert2');
    });
    NovComet.run();
});
        </script>
    </body>
</html>
```

The client code:

```php
//NovComet.js
NovComet = {
    sleepTime: 1000,
    _subscribed: {},
    _timeout: undefined,
    _baseurl: "comet.php",
    _args: '',
    _urlParam: 'subscribed',
 
    subscribe: function(id, callback) {
        NovComet._subscribed[id] = {
            cbk: callback,
            timestamp: NovComet._getCurrentTimestamp()
        };
        return NovComet;
    },
 
    _refresh: function() {
        NovComet._timeout = setTimeout(function() {
            NovComet.run()
        }, NovComet.sleepTime);
    },
 
    init: function(baseurl) {
        if (baseurl!=undefined) {
            NovComet._baseurl = baseurl;
        }
    },
 
    _getCurrentTimestamp: function() {
        return Math.round(new Date().getTime() / 1000);
    },
 
    run: function() {
        var cometCheckUrl = NovComet._baseurl + '?' + NovComet._args;
        for (var id in NovComet._subscribed) {
            var currentTimestamp = NovComet._subscribed[id]['timestamp'];
 
            cometCheckUrl += '&' + NovComet._urlParam+ '[' + id + ']=' +
               currentTimestamp;
        }
        cometCheckUrl += '&' + NovComet._getCurrentTimestamp();
        $.getJSON(cometCheckUrl, function(data){
            switch(data.s) {
                case 0: // sin cambios
                    NovComet._refresh();
                    break;
                case 1: // trigger
                    for (var id in data['k']) {
                        NovComet._subscribed[id]['timestamp'] = data['k'][id];
                        NovComet._subscribed[id].cbk(data.k);
                    }
                    NovComet._refresh();
                    break;
            }
        });
 
    },
 
    publish: function(id) {
        var cometPublishUrl = NovComet._baseurl + '?' + NovComet._args;
        cometPublishUrl += '&publish=' + id;
        $.getJSON(cometPublishUrl);
    }
};
```

The server-side PHP

```php
// comet.php
include('NovComet.php');
 
$comet = new NovComet();
$publish = filter_input(INPUT_GET, 'publish', FILTER_SANITIZE_STRING);
if ($publish != '') {
    echo $comet->publish($publish);
} else {
    foreach (filter_var_array($_GET['subscribed'], FILTER_SANITIZE_NUMBER_INT) as $key => $value) {
        $comet->setVar($key, $value);
    }
    echo $comet->run();
}
```

and my comet library implementation:

```php
// NovComet.php
class NovComet {
    const COMET_OK = 0;
    const COMET_CHANGED = 1;
 
    private $_tries;
    private $_var;
    private $_sleep;
    private $_ids = array();
    private $_callback = null;
 
    public function  __construct($tries = 20, $sleep = 2)
    {
        $this->_tries = $tries;
        $this->_sleep = $sleep;
    }
 
    public function setVar($key, $value)
    {
        $this->_vars[$key] = $value;
    }
 
    public function setTries($tries)
    {
        $this->_tries = $tries;
    }
 
    public function setSleepTime($sleep)
    {
        $this->_sleep = $sleep;
    }
 
    public function setCallbackCheck($callback)
    {
        $this->_callback = $callback;
    }
 
    const DEFAULT_COMET_PATH = "/dev/shm/%s.comet";
 
    public function run() {
        if (is_null($this->_callback)) {
            $defaultCometPAth = self::DEFAULT_COMET_PATH;
            $callback = function($id) use ($defaultCometPAth) {
                $cometFile = sprintf($defaultCometPAth, $id);
                return (is_file($cometFile)) ? filemtime($cometFile) : 0;
            };
        } else {
            $callback = $this->_callback;
        }
 
        for ($i = 0; $i < $this->_tries; $i++) {
            foreach ($this->_vars as $id => $timestamp) {
                if ((integer) $timestamp == 0) {
                    $timestamp = time();
                }
                $fileTimestamp = $callback($id);
                if ($fileTimestamp > $timestamp) {
                    $out[$id] = $fileTimestamp;
                }
                clearstatcache();
            }
            if (count($out) > 0) {
                return json_encode(array('s' => self::COMET_CHANGED, 'k' => $out));
            }
            sleep($this->_sleep);
        }
        return json_encode(array('s' => self::COMET_OK));
    }
 
    public function publish($id)
    {
        return json_encode(touch(sprintf(self::DEFAULT_COMET_PATH, $id)));
    }
}
```

As you can see in my example I’ve created a personal protocol for the communications between the client (js at browser), and the server (PHP). It’s a simple one. If you’re looking for a “standard” protocol maybe you need have a look to [bayeux](http://svn.cometd.com/trunk/bayeux/bayeux.html) protocol from Dojo people.

Let me explain a little bit the usage of the script:

- In the HTML page we start the listener (NovComet.subscribe).
- We can subscribe to as many events we want (OK it depends on our resources)
- When we subscribe to one event we pass a callback function to be triggered.
- When we subscribe to the event, we pass the current timestamp to the server.
- Client side script (js with jquery) will call to server-side script (PHP) with the timestamp and will wait until server finish.
- Server side script will answer when even timestamp changes (someone has published the event)
- Server side will no keep waiting forever. If nobody publish the event, server will answer after a pre-selected timeout
- client side script will repeat the process again and again.

There’s something really important with this technique. Our server-side event check need to be as simpler as we can. We cannot execute a SQL query for example (our sysadmin will kill us if we do it). We need to bear in mind that this check will be performed again and again per user, because of that it must be as light as we can. In this example we are checking the last modification date of a file ([filemtime](http://php.net/manual/en/function.filemtime.php)). Another good solution is to use a [memcached](http://php.net/manual/en/book.memcache.php) database and check a value.

For the test I’ve also created a publishing script (NovComet.publish). This is the simple part. We only call a server-side script that touch the event file (changing the last modification date), triggering the event.

[](/assets/images/firebug.png "firebug")

Now I'm going to explain what we can see on the firebug console:

1. The first iteration nothing happens. 200 OK Http code after the time-out set in the PHP script
2. As we can see here the script returns a JSON with s=0 (nothing happens)
3. Now we publish an event. Script returns a 200 OK but now the JSON is different. s=1 and the time-stamp of the event
4. Our callback has been triggered
5. And next iteration waiting

And that’s all. Simple and useful. But remember, you must take care if you are using this solution within a high traffic site. What do you think? Do you use lazy comet with PHP in production servers or would you rather another solution?

You can get the code at github [here](https://github.com/gonzalo123/nov-comet).
