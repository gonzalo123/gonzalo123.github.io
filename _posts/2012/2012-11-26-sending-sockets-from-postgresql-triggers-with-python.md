---
layout: post
title: "Sending sockets from PostgreSQL triggers with Python"
date: "2012-11-26"
tags: 
  - "node-js"
  - "php"
  - "postgresql"
  - "python"
  - "technology"
  - "nodejs"
  - "postgresql"
---

Picture this: We want to notify to one external service each time that one record is inserted in the database. We can find the place where the insert statement is done and create a TCP client there, but: What happens if the application that inserts the data within the database is a legacy application?, or maybe it is too hard to do?. If your database is PostgreSQL it's pretty straightforward. With the "[default](http://www.postgresql.org/docs/9.2/static/plpgsql.html)" procedural language of PostgreSQL (**pgplsql**) we cannot do it, but PostgreSQL allows us to use more procedural languages than plpgsql, for example [Python](http://www.postgresql.org/docs/9.2/static/plpython.html). With **plpython** we can use sockets in the same way than we use it within Python scripts. It's very simple. Let me show you how to do it.

First we need to create one plpython with our TCP client

```postgresql
CREATE OR REPLACE FUNCTION dummy.sendsocket(msg character varying, host character varying, port integer)
  RETURNS integer AS
$BODY$
  import _socket
  try:
    s = _socket.socket(_socket.AF_INET, _socket.SOCK_STREAM)
    s.connect((host, port))
    s.sendall(msg)
    s.close()
    return 1
  except:
    return 0
$BODY$
  LANGUAGE plpython VOLATILE
  COST 100;
ALTER FUNCTION dummy.sendsocket(character varying, character varying, integer)
  OWNER TO username;
```

Now we create the trigger that use our socket client.

```postgresql
CREATE OR REPLACE FUNCTION dummy.myTriggerToSendSockets()
RETURNS trigger AS
$BODY$
   import json
   stmt = plpy.prepare("select dummy.sendSocket($1, $2, $3)", ["text", "text", "int"])
   rv = plpy.execute(stmt, [json.dumps(TD), "host", 26200])
$BODY$
LANGUAGE plpython VOLATILE
COST 100;
```

As you can see in my example we are sending all the record as a JSON string in the socket body.

And finally we attach the trigger to one table (or maybe we need to do it to more than one table)

```postgresql
CREATE TRIGGER myTrigger
  AFTER INSERT OR UPDATE OR DELETE
  ON dummy.myTable
  FOR EACH ROW
  EXECUTE PROCEDURE dummy.myTriggerToSendSockets();
```

And that's all. Now we can use one simple TCP socket server to handle those requests. Let me show you different examples of TCP servers with different languages. As we can see all are different implementations of [Reactor](http://en.wikipedia.org/wiki/Reactor_pattern) pattern. We can use, for example:

**node.js:** 

```javascript
var net = require('net');
 
var host = 'localhost';
var port = 26200;
 
var server = net.createServer(function (socket) {
    socket.on('data', function(buffer) {
        // do whatever that we want with buffer
    });
});
 
server.listen(port, host);
```

**python (with [Twisted](http://twistedmatrix.com/trac/)):** 

```python
from twisted.internet import reactor, protocol
 
HOST = 'localhost'
PORT = 26200
 
class MyServer(protocol.Protocol):
    def dataReceived(self, data):
        # do whatever that we want with data
        pass
 
class MyServerFactory(protocol.Factory):
    def buildProtocol(self, addr):
        return MyServer()
 
reactor.listenTCP(PORT, MyServerFactory(), interface=HOST)
reactor.run()
```

(I know that we can create the Python's TCP server without Twisted, but if don't use it maybe someone will angry with me. Probably he is angry right now because I put the node.js example first :))

**php (with [react](https://github.com/reactphp/react)):** 
```php
<?php
include __DIR__ . '/vendor/autoload.php';
 
$host = 'localhost';
$port = 26200;
 
$loop   = React\EventLoop\Factory::create();
$socket = new React\Socket\Server($loop);
 
$socket->on('connection', function ($conn) {
    $conn->on('data', function ($data) {
        // do whatever we want with data
        }
    );
});
 
$socket->listen($port, $host);
$loop->run();
```

You also can use [xinet.d](http://gonzalo123.com/2010/05/23/building-network-services-with-php-and-xinetd/ "Building network services with PHP and xinetd") to handle the TCP inbound connections.
