---
layout: post
title: "Notify events from PostgreSQL to external listeners"
date: "2016-07-04"
tags: 
  - "js"
  - "node-js"
  - "technology"
  - "listen"
  - "nodejs"
  - "notify"
  - "postgresql"
---

Sometimes we need to call external programs from our PostgreSQL database. We can send sockets from SQL statements. I've written about [it](https://gonzalo123.com/2012/11/26/sending-sockets-from-postgresql-triggers-with-python/). The problem with this approach the following one. If user rollbacks the transaction the socket has been already emitted. That's a problem (or not. Depending on our application). Nobody also guarantees that the process behind the socket server has access to the data of the transaction. If we're very fast maybe the transaction isn't commited yet. We can use one sleep function but sleep functions are always a bad idea. PostgreSQL gives us another tool to decouple processes: [LISTEN and NOTIFY](https://www.postgresql.org/docs/9.1/static/sql-notify.html).

Let me show you and example. First we create a table:

```postgresql
CREATE TABLE PUBLIC.TBLEXAMPLE
(
  KEY1 CHARACTER VARYING(10) NOT NULL,
  KEY2 CHARACTER VARYING(14) NOT NULL,
 
  VALUE1 CHARACTER VARYING(20),
  VALUE2 CHARACTER VARYING(20) NOT NULL,
 
  CONSTRAINT TBLEXAMPLE_PKEY PRIMARY KEY (KEY1, KEY2)
)
```

Now we add a "after insert" trigger to our table

```postgresql
CREATE TRIGGER TBLEXAMPLE_AFTER
AFTER INSERT
ON PUBLIC.TBLEXAMPLE
FOR EACH ROW
EXECUTE PROCEDURE PUBLIC.NOTIFY();
```

And now, within the trigger function, we send a notify event ('myEvent' in this case) with the row information. We need to send plain text in the notify event so we'll use JSON to encode our row data.

```postgresql
CREATE OR REPLACE FUNCTION PUBLIC.NOTIFY() RETURNS trigger AS
$BODY$
BEGIN
  PERFORM pg_notify('myEvent', row_to_json(NEW)::text);
  RETURN new;
END;
$BODY$
LANGUAGE 'plpgsql' VOLATILE COST 100;
```

Now we're going to build a server side example that connects to our PostgreSQL database and listen to the event. In this case we're going to use nodejs to build the prototype. This example also will enqueue events into a gearman server.

```javascript
var pg = require('pg'),
    gearmanode = require('gearmanode'),
    gearmanClient,
    conString = 'tcp://username:password@localhost:5432/gonzalo',
    pgClient;
 
gearmanode.Client.logger.transports.console.level = 'error';
 
gearmanClient = gearmanode.client();
 
console.log('LISTEN myEvent');
pgClient = new pg.Client(conString);
pgClient.connect();
pgClient.query('LISTEN myEvent');
pgClient.on('notification', function (data) {
    console.log("\033[34m" + new Date + '-\033[0m payload', data.payload);
    gearmanClient.submitJob('sms.sender.one', data.payload);
});
```

And that's all. Now we only need to perform an INSERT statement into our table. This process will trigger our event and our nodejs will enqueue the process into a gearman queue.

```postgresql
INSERT INTO PUBLIC.TBLEXAMPLE(KEY1, KEY2, VALUE1, VALUE2) VALUES ('k1', 'k2', 'v1', 'v2');
```

It's good to remark that if our insert statement is inside a transaction and we rollback it, notify won't send any message.
