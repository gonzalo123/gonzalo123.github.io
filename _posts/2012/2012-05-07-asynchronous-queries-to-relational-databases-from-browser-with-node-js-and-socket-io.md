---
layout: post
title: "Asynchronous queries to PostgreSql database from the browser with node.js and socket.io"
date: "2012-05-07"
tags: 
  - "node-js"
  - "postgresql"
  - "socket-io"
  - "technology"
---

Normally we perform our database connection at server side with PHP and PDO for example. In this post I will show a simple technique to send queries to our database (PostgreSql in this example) asynchronously using node.js and socket.io.

The idea is pretty straightforward. We will send the SQL string and the values with a WebSocket and we will execute a callback in the client when the server (node.js script) fetches the recordset.

Our server:

```javascript
var pg = require('pg');
 
var conString = "tcp://user:password@localhost:5432/db";
var client = new pg.Client(conString);
client.connect();
 
var io = require('socket.io').listen(8888);
 
io.sockets.on('connection', function (socket) {
    socket.on('sql', function (data) {
        var query = client.query(data.sql, data.values);
        query.on('row', function(row) {
            socket.emit('sql', row);
        });
    });
});
```

And our client:

```html
<script src="http://localhost:8888/socket.io/socket.io.js"></script>
<script>
var socket = io.connect('http://vmserver:8888');
 
function sendSql(sql, values, cbk) {
    socket.emit('sql', { sql: sql, values : values});
    socket.on('sql', function(data){
        console.log(data);
    });
}
</script>    
<p>
<a href="#" onclick="sendSql('select * from users', [], function(data) {console.log(data);})">select * from users</a>
</p>
<p>
<a href="#" onclick="sendSql('select * from users where uid=$1', [4], function(data) {console.log(data);})">select * from users where uid=$1</a>
</p>
```

Simple, isn't it? You must take care if you use this script at production. Our database is exposed to raw SQL sent from the client. It's a concept example. Maybe it would be better not to send the SQL. Store them into key-value table in the server and send only an ID from the browser.

What do you think?
