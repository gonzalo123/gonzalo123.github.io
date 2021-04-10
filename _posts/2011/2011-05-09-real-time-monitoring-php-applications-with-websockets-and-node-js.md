---
layout: post
title: "Real time monitoring PHP applications with websockets and node.js"
date: "2011-05-09"
tags: 
  - "html5"
  - "node-js"
  - "php"
  - "technology"
  - "websockets"
  - "html5"
  - "node"
  - "nodejs"
  - "websockets"
---

The inspection of the error logs is a common way to detect errors and bugs. We also can show errors on-screen within our developement server, or we even can use great tools like [firePHP](http://www.firephp.org/) to show our PHP errors and warnings inside our firebug console. That’s cool, but we only can see our session errors/warnings. If we want to see another's errors we need to inspect the error log. _tail -f_ is our friend, but we need to surf against all the warnings of all sessions to see our desired ones. Because of that I want to build a tool to monitor my PHP applications in real-time. Let’s start:

What’s the idea? The idea is catch all PHP’s errors and warnings at run time and send them to a [node.js](http://nodejs.org/) HTTP server. This server will work similar than a chat server but our clients will only be able to read the server’s logs. Basically the applications have three parts: the node.js server, the web client (html5) and the server part (PHP). Let me explain a bit each part:

## The node Server

Basically it has two parts: a http server to handle the PHP errors/warnings and a [websocket](http://dev.w3.org/html5/websockets/) server to manage the realtime communications with the browser. When I say that I’m using websockets that’s means the web client will only work with a browser with websocket support like chrome. Anyway it’s pretty straightforward swap from a websocket sever to a [socket.io](http://socket.io/) server to use it with every browser. But websockets seems to be the future, so I will use websockets in this example.

**The http server:** 

```javascript
http.createServer(function (req, res) {
    var remoteAdrress = req.socket.remoteAddress;
    if (allowedIP.indexOf(remoteAdrress) >= 0) {
        res.writeHead(200, {
            'Content-Type': 'text/plain'
        });
        res.end('Ok\n');
        try {
            var parsedUrl = url.parse(req.url, true);
            var type = parsedUrl.query.type;
            var logString = parsedUrl.query.logString;
            var ip = eval(parsedUrl.query.logString)[0];
            if (inspectingUrl == "" ||  inspectingUrl == ip) {
                clients.forEach(function(client) {
                    client.write(logString);
                });
            }
        } catch(err) {
            console.log("500 to " + remoteAdrress);
            res.writeHead(500, {
                'Content-Type': 'text/plain'
            });
            res.end('System Error\n');
        }
    } else {
        console.log("401 to " + remoteAdrress);
        res.writeHead(401, {
            'Content-Type': 'text/plain'
        });
        res.end('Not Authorized\n');
    }
}).listen(httpConf.port, httpConf.host);
```

and **the web socket server:**

```javascript
var inspectingUrl = undefined;
 
ws.createServer(function(websocket) {
    websocket.on('connect', function(resource) {
        var parsedUrl = url.parse(resource, true);
        inspectingUrl = parsedUrl.query.ip;
        clients.push(websocket);
    });
 
    websocket.on('close', function() {
        var pos = clients.indexOf(websocket);
        if (pos >= 0) {
            clients.splice(pos, 1);
        }
    });
 
}).listen(wsConf.port, wsConf.host);
```

If you want to know more about node.js and see more examples, have a look to the great site: [http://nodetuts.com/](http://nodetuts.com/). In this site Pedro Teixeira will show examples and node.js tutorials. In fact my node.js http + websoket server is a mix of two tutorials from this site.

## The web client.

The web client is a simple websockets application. We will handle the websockets connection, reconnect if it dies and a bit more. I’s based on node.js chat [demo](http://chat.nodejs.org/)

```php
<?php $ip = filter_input(INPUT_GET, 'ip', FILTER_SANITIZE_STRING); ?>
 
        Real time <?= $ip ?> monitor
<script type="text/javascript" src="http://ajax.googleapis.com/ajax/libs/jquery/1.4.3/jquery.min.js"></script><script type="text/javascript">// <![CDATA[
            selectedIp = '<?= $ip ?>';
 
// ]]></script>
<script type="text/javascript" src="js.js"></script>
</pre>
<div id="toolbar">
<ul id="status">
    <li>Socket status: <span id="socketStatus">Conecting ...</span></li>
    <li>IP: <!--?= $ip == '' ? 'all' : $ip . " <a href='?ip='-->[all]" ?></li>
    <li>count: <span id="count">0</span></li>
</ul>
</div>
<pre>
```

And the **javascript magic** 

```javascript
var timeout = 5000;
var wsServer = '192.168.2.2:8880';
var unread = 0;
var focus = false;
 
var count = 0;
function updateCount() {
    count++;
    $("#count").text(count);
}
 
function cleanString(string) {
    return string.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
}
 
function updateUptime () {
    var now = new Date();
    $("#uptime").text(now.toRelativeTime());
}
 
function updateTitle(){
    if (unread) {
        document.title = "(" + unread.toString() + ") Real time " + selectedIp + " monitor";
    } else {
        document.title = "Real time " + selectedIp + " monitor";
    }
}
 
function pad(n) {
    return ("0" + n).slice(-2);
}
 
function startWs(ip) {
    try {
        ws = new WebSocket("ws://" + wsServer + "?ip=" + ip);
        $('#toolbar').css('background', '#65A33F');
        $('#socketStatus').html('Connected to ' + wsServer);
        //console.log("startWs:" + ip);
        //listen for browser events so we know to update the document title
        $(window).bind("blur", function() {
            focus = false;
            updateTitle();
        });
 
        $(window).bind("focus", function() {
            focus = true;
            unread = 0;
            updateTitle();
        });
    } catch (err) {
        //console.log(err);
        setTimeout(startWs, timeout);
    }
 
    ws.onmessage = function(event) {
        unread++;
        updateTitle();
        var now = new Date();
        var hh = pad(now.getHours());
        var mm = pad(now.getMinutes());
        var ss = pad(now.getSeconds());
 
        var timeMark = '[' + hh + ':' + mm + ':' + ss + '] ';
        logString = eval(event.data);
        var host = logString[0];
        var line = "<table class='message'><tr><td width='1%' class='date'>" + timeMark + "</td><td width='1%' valign='top' class='host'><a href=?ip=" + host + ">" + host + "</a></td>";
        line += "<td class='msg-text' width='98%'>" + logString[1]; + "</td></tr>";
        if (logString[2]) {
            line += "<tr><td>&nbsp;</td><td colspan='3' class='msg-text'>" + logString[2] + "</td></tr>";
        }
 
        $('#log').append(line);
        updateCount();
        window.scrollBy(0, 100000000000000000);
    };
 
    ws.onclose = function(){
        //console.log("ws.onclose");
        $('#toolbar').css('background', '#933');
        $('#socketStatus').html('Disconected');
        setTimeout(function() {startWs(selectedIp)}, timeout);
    }
}
 
$(document).ready(function() {
    startWs(selectedIp);
});
```

## The server part:

The server part will handle silently all PHP warnings and errors and it will send them to the node server. The idea is to place a minimal PHP line of code at the beginning of the application that we want to monitor. Imagine the following piece of PHP code

```php
$a = $var[1];
$a = 1/0;
class Dummy
{
    static function err()
    {
        throw new Exception("error");
    }
}
Dummy1::err();
```

it will throw: A notice: Undefined variable: var A warning: Division by zero An Uncaught exception 'Exception' with message 'error'

So we will add our small library to catch those errors and send them to the node server

```php
include('client/NodeLog.php');
NodeLog::init('192.168.2.2');
 
$a = $var[1];
$a = 1/0;
class Dummy
{
    static function err()
    {
        throw new Exception("error");
    }
}
Dummy1::err();
```

The script will work in the same way than the fist version but if we start our node.js server in a console:

```commandline
$ node server.js
HTTP server started at 192.168.2.2::5672
Web Socket server started at 192.168.2.2::8880
```

We will see those errors/warnings in real-time when we start our browser

Here we can see a small screencast with the working application:


[![youtube](https://img.youtube.com/vi/KnBRcU\_tBkk\/0.jpg)](https://www.youtube.com/watch?v=KnBRcU\_tBkk\)

This is the server side library:

```php
class NodeLog
{
    const NODE_DEF_HOST = '127.0.0.1';
    const NODE_DEF_PORT = 5672;
 
    private $_host;
    private $_port;
 
    /**
     * @param String $host
     * @param Integer $port
     * @return NodeLog
     */
    static function connect($host = null, $port = null)
    {
        return new self(is_null($host) ? self::$_defHost : $host, is_null($port) ? self::$_defPort : $port);
    }
 
    function __construct($host, $port)
    {
        $this->_host = $host;
        $this->_port = $port;
    }
 
    /**
     * @param String $log
     * @return Array array($status, $response)
     */
    public function log($log)
    {
        list($status, $response) = $this->send(json_encode($log));
        return array($status, $response);
    }
 
    private function send($log)
    {
        $url = "http://{$this->_host}:{$this->_port}?logString=" . urlencode($log);
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_NOBODY, true);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
 
        $response = curl_exec($ch);
        $status   = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
 
        return array($status, $response);
    }
 
    static function getip() {
        $realip = '0.0.0.0';
        if ($_SERVER) {
            if ( isset($_SERVER['HTTP_X_FORWARDED_FOR']) && $_SERVER['HTTP_X_FORWARDED_FOR'] ) {
                $realip = $_SERVER["HTTP_X_FORWARDED_FOR"];
            } elseif ( isset($_SERVER['HTTP_CLIENT_IP']) && $_SERVER["HTTP_CLIENT_IP"] ) {
                $realip = $_SERVER["HTTP_CLIENT_IP"];
            } else {
                $realip = $_SERVER["REMOTE_ADDR"];
            }
        } else {
            if ( getenv('HTTP_X_FORWARDED_FOR') ) {
                $realip = getenv('HTTP_X_FORWARDED_FOR');
            } elseif ( getenv('HTTP_CLIENT_IP') ) {
                $realip = getenv('HTTP_CLIENT_IP');
            } else {
                $realip = getenv('REMOTE_ADDR');
            }
        }
        return $realip;
    }
 
    public static function getErrorName($err)
    {
        $errors = array(
            E_ERROR             => 'ERROR',
            E_RECOVERABLE_ERROR => 'RECOVERABLE_ERROR',
            E_WARNING           => 'WARNING',
            E_PARSE             => 'PARSE',
            E_NOTICE            => 'NOTICE',
            E_STRICT            => 'STRICT',
            E_DEPRECATED        => 'DEPRECATED',
            E_CORE_ERROR        => 'CORE_ERROR',
            E_CORE_WARNING      => 'CORE_WARNING',
            E_COMPILE_ERROR     => 'COMPILE_ERROR',
            E_COMPILE_WARNING   => 'COMPILE_WARNING',
            E_USER_ERROR        => 'USER_ERROR',
            E_USER_WARNING      => 'USER_WARNING',
            E_USER_NOTICE       => 'USER_NOTICE',
            E_USER_DEPRECATED   => 'USER_DEPRECATED',
        );
        return $errors[$err];
    }
 
    private static function set_error_handler($nodeHost, $nodePort)
    {
        set_error_handler(function ($errno, $errstr, $errfile, $errline) use($nodeHost, $nodePort) {
            $err = NodeLog::getErrorName($errno);
            /*
            if (!(error_reporting() & $errno)) {
                // This error code is not included in error_reporting
                return;
            }
            */
            $log = array(
                NodeLog::getip(),
                "<strong class="{$err}">{$err}</strong> {$errfile}:{$errline}",
                nl2br($errstr)
            );
            NodeLog::connect($nodeHost, $nodePort)->log($log);
            return false;
        });
    }
 
    private static function register_exceptionHandler($nodeHost, $nodePort)
    {
        set_exception_handler(function($exception) use($nodeHost, $nodePort) {
            $exceptionName = get_class($exception);
            $message = $exception->getMessage();
            $file = $exception->getFile();
            $line = $exception->getLine();
            $trace = $exception->getTraceAsString();
 
            $msg = count($trace) > 0 ? "Stack trace:\n{$trace}" : null;
            $log = array(
                NodeLog::getip(),
                nl2br("<strong class="ERROR">Uncaught exception '{$exceptionName}'</strong> with message '{$message}' in {$file}:{$line}"),
                nl2br($msg)
            );
            NodeLog::connect($nodeHost, $nodePort)->log($log);
            return false;
        });
    }
 
    private static function register_shutdown_function($nodeHost, $nodePort)
    {
        register_shutdown_function(function() use($nodeHost, $nodePort) {
            $error = error_get_last();
 
            if ($error['type'] == E_ERROR) {
                $err = NodeLog::getErrorName($error['type']);
                $log = array(
                    NodeLog::getip(),
                    "<strong class="{$err}">{$err}</strong> {$error['file']}:{$error['line']}",
                    nl2br($error['message'])
                );
                NodeLog::connect($nodeHost, $nodePort)->log($log);
            }
            echo NodeLog::connect($nodeHost, $nodePort)->end();
        });
    }
 
    private static $_defHost = self::NODE_DEF_HOST;
    private static $_defPort = self::NODE_DEF_PORT;
 
    /**
     * @param String $host
     * @param Integer $port
     * @return NodeLog
     */
    public static function init($host = self::NODE_DEF_HOST, $port = self::NODE_DEF_PORT)
    {
        self::$_defHost = $host;
        self::$_defPort = $port;
 
        self::register_exceptionHandler($host, $port);
        self::set_error_handler($host, $port);
        self::register_shutdown_function($host, $port);
 
        $node = self::connect($host, $port);
        $node->start();
        return $node;
    }
 
    private static $time;
    private static $mem;
 
    public function start()
    {
        self::$time = microtime(TRUE);
        self::$mem = memory_get_usage();
        $log = array(NodeLog::getip(), "<strong class="OK">Start</strong> >>>> {$_SERVER['REQUEST_URI']}");
        $this->log($log);
    }
 
    public function end()
    {
        $mem = (memory_get_usage() - self::$mem) / (1024 * 1024);
        $time = microtime(TRUE) - self::$time;
        $log = array(NodeLog::getip(), "<strong class="OK">End</strong> <<<< mem: {$mem} time {$time}");         $this->log($log);
    }
}
```

And of course the full code on gitHub: [RealTimeMonitor](https://github.com/gonzalo123/RealTimeMonitor)
