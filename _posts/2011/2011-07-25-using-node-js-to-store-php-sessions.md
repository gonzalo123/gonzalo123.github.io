---
layout: post
title: "Using node.js to store PHP sessions"
date: "2011-07-25"
tags: 
  - "node-js"
  - "npm"
  - "php"
  - "technology"
  - "node"
  - "nodeks"
  - "sessions"
---

We use [sessions](http://www.php.net/manual/en/book.session.php) when we want to preserve certain data across subsequent accesses. PHP allows us to use different handlers when we're using sessions. The default one is filesystem, but we can change it with [session.save\_handler](http://www.php.net/manual/en/session.configuration.php#ini.session.save-handler) in the php.ini. session.save\_handler defines the name of the handler which is used for storing and retrieving data associated with a session. We also can create our own handler to manage sessions. In this post we're going to create a custom handler to store sessions in a node.js service. Let's start:

Imagine we've got the following php script:

```php
session_start();
 
if (!isset($_SESSION["gonzalo"])) $_SESSION["gonzalo"] = 0;
$_SESSION["gonzalo"]++;
$_SESSION["arr"] = array('key' => uniqid());
var_dump($_SESSION);
```

A simple usage of sessions with PHP. If we reload the page our counter will be incremented by one. We're using the default session handler. It works without any problem.

The idea is create a custom handler to use a server with node.js to store the session information instead of filesystem. To create custom handlers we need to use the PHP function: [session\_set\_save\_handler](http://www.php.net/manual/en/function.session-set-save-handler.php) and rewrite the callbacks for: open, close, read, write, destroy and gc. PHP's documentation is great. My proposal is the following one:

Our custom handler: 

```php
class NodeSession
{
    const NODE_DEF_HOST = '127.0.0.1';
    const NODE_DEF_PORT = 5672;
 
    static function start($host = self::NODE_DEF_HOST, $port = self::NODE_DEF_PORT)
    {
        $obj = new self($host, $port);
        session_set_save_handler(
            array($obj, "open"),
            array($obj, "close"),
            array($obj, "read"),
            array($obj, "write"),
            array($obj, "destroy"),
            array($obj, "gc"));
        session_start();
        return $obj;
    }
 
    private function unserializeSession($data)
    {
        if(  strlen( $data) == 0) {
            return array();
        }
 
        // match all the session keys and offsets
        preg_match_all('/(^|;|\})([a-zA-Z0-9_]+)\|/i', $data, $matchesarray, PREG_OFFSET_CAPTURE);
        $returnArray = array();
 
        $lastOffset = null;
        $currentKey = '';
        foreach ( $matchesarray[2] as $value ) {
            $offset = $value[1];
            if(!is_null( $lastOffset)) {
                $valueText = substr($data, $lastOffset, $offset - $lastOffset );
                $returnArray[$currentKey] = unserialize($valueText);
            }
            $currentKey = $value[0];
 
            $lastOffset = $offset + strlen( $currentKey )+1;
        }
 
        $valueText = substr($data, $lastOffset );
        $returnArray[$currentKey] = unserialize($valueText);
 
        return $returnArray;
    }
     
    function __construct($host = self::NODE_DEF_HOST, $port = self::NODE_DEF_PORT)
    {
        $this->_host = $host;
        $this->_port = $port;
    }
 
    function open($save_path, $session_name)
    {
        return true;
    }
 
    function close()
    {
        return true;
    }
 
    public function read($id)
    {
        return (string) $this->send(__FUNCTION__, array('id' => $id));
    }
 
    public function write($id, $data)
    {
        try {
            $this->send(__FUNCTION__, array(
                'id'       => $id,
                'data'     => $data,
                'time'     => time(),
                'dataJSON' => json_encode($this->unserializeSession($data))));
            return true;
        } catch (Exception $e) {
            return false;
        }
    }
 
    public function destroy($id)
    {
        try {
            $this->send(__FUNCTION__, array('id' => $id));
        } catch (Exception $e) {
            return false;
        }
         return true;
    }
 
    function gc($maxlifetime)
    {
        try {
            $this->send(__FUNCTION__, array('maxlifetime' => $maxlifetime, 'time' => time()));
        } catch (Exception $e) {
            return false;
        }
        return true;
    }
 
    private function send($action, $params)
    {
        $params = array('action' => $action) + $params;
        return file_get_contents("http://{$this->_host}:{$this->_port}?" . http_build_query($params));
    }
}
```

Our node.js server: 

```javascript
var http = require('http'),
    url  = require('url'),
    session = require('nodePhpSessions').SessionHandler;
 
var sessionHandler = new session();
 
var server = http.createServer(function (req, res) {
    var parsedUrl = url.parse(req.url, true).query;
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end(sessionHandler.run(parsedUrl));
});
 
server.listen(5672, "127.0.0.1", function() {
  var address = server.address();
  console.log("opened server on %j", address);
});
```

As we can see we need the node.js module nodePhpSessions. You can easily install with: 

```commandline
npm install nodePhpSessions
```

You can see nodePhpSessions library [here](https://github.com/gonzalo123/nodePhpSessions/blob/master/lib/SessionHandler.js).

The library is tested with nodeunit. Without TDD is very hard to test things such as garbage collector.: 

```javascript
var session = require('nodePhpSessions').SessionHandler;
var sessionHandler = new session();
var parsedUrl;
 
exports["testReadUndefinedSession"] = function(test){
    parsedUrl = { action: 'read', id: 'ts49vmf0p732iafr25mdu8gvg2' };
    test.equal(sessionHandler.run(parsedUrl), undefined);
    test.done();
};
 
exports["oneSessionShouldReturns1"] = function(test){
    parsedUrl = {
        action: 'write',
        id: 'ts49vmf0p732iafr25mdu8gvg2',
        data: 'gonzalo|i:1;arr|a:1:{s:3:"key";s:13:"4e2b1a40d136a";}',
        time: '1311447616',
        dataJSON: '{"gonzalo":1,"arr":{"key":"4e2b1a40d136a"}}' };
    sessionHandler.run(parsedUrl);
 
    parsedUrl = { action: 'readAsArray', id: 'ts49vmf0p732iafr25mdu8gvg2' };
    test.equal(sessionHandler.run(parsedUrl).gonzalo, 1);
    test.done();
};
 
exports["oneSessionShouldReturns2"] = function(test){
    parsedUrl = {
        action: 'write',
        id: 'ts49vmf0p732iafr25mdu8gvg2',
        data: 'gonzalo|i:2;arr|a:1:{s:3:"key";s:13:"4e2b1a40d136a";}',
        time: '1311447616',
        dataJSON: '{"gonzalo":2,"arr":{"key":"4e2b1a40d136a"}}' };
    sessionHandler.run(parsedUrl);
    parsedUrl = { action: 'readAsArray', id: 'ts49vmf0p732iafr25mdu8gvg2' };
    test.equal(sessionHandler.run(parsedUrl).gonzalo, 2);
    test.done();
};
 
exports["destroySession"] = function(test){
    parsedUrl = {
        action: 'destroy',
        id: 'ts49vmf0p732iafr25mdu8gvg2'};
    sessionHandler.run(parsedUrl);
 
    parsedUrl = { action: 'readAsArray', id: 'ts49vmf0p732iafr25mdu8gvg2' };
    test.equal(sessionHandler.run(parsedUrl), undefined);
 
    test.done();
};
 
exports["garbageColector"] = function(test){
    parsedUrl = {
        action: 'write',
        id: 'session1',
        data: 'gonzalo|i:1;arr|a:1:{s:3:"key";s:13:"4e2b1a40d136a";}',
        time: '1111111200',
        dataJSON: '{"gonzalo":1,"arr":{"key":"4e2b1a40d136a"}}' };
    sessionHandler.run(parsedUrl);
 
    parsedUrl = {
        action: 'write',
        id: 'session2',
        data: 'gonzalo|i:1;arr|a:1:{s:3:"key";s:13:"4e2b1a40d136a";}',
        time: '1111111100',
        dataJSON: '{"gonzalo":1,"arr":{"key":"4e2b1a40d136a"}}' };
    sessionHandler.run(parsedUrl);
 
    parsedUrl = { action: 'gc', maxlifetime: '100', time: '1111111210'};
    sessionHandler.run(parsedUrl);
 
    parsedUrl = { action: 'readAsArray', id: 'session2' };
    test.equal(sessionHandler.run(parsedUrl), undefined);
 
    parsedUrl = { action: 'readAsArray', id: 'session1' };
    test.equal(sessionHandler.run(parsedUrl).gonzalo, 1);
 
    test.done();
};
```

Here you can see the output of the tests: 

```commandline
nodeunit testNodeSessions.js 
 
testNodeSessions.js
✔ testReadUndefinedSession
✔ oneSessionShouldReturns1
✔ oneSessionShouldReturns2
✔ destroySession
✔ garbageColector
 
OK: 6 assertions (5ms)
```

Now we change the original PHP script to: 

```php
include_once 'NodeSessions.php';
NodeSession::start();
 
if (!isset($_SESSION["gonzalo"])) $_SESSION["gonzalo"] = 0;
$_SESSION["gonzalo"]++;
$_SESSION["arr"] = array('key' => uniqid());
var_dump($_SESSION);
```

We start the node.js server:

```commandline
node serverSessions.js
```

Now if we reload our script in the browser we will see the same behaviour, but now our sessions are stored in the node.js server.

```php
array(2) {
  ["gonzalo"]=>
  int(16)
  ["arr"]=>
  array(1) {
    ["key"]=>
    string(13) "4e2a9f6a966f4"
  }
}
```

This kind of techniques are good when [clustering PHP applications](http://gonzalo123.wordpress.com/2010/07/27/clustering-php-applications-tips-and-hints/).

Full code is available on github (node server, PHP handler, tests and examples) [here](https://github.com/gonzalo123/nodePhpSessions).
