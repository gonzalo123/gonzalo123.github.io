---
layout: post
title: "Building one HTTP client in PostgreSQL with PL/Python"
date: "2015-09-21"
tags: 
  - "pl-python"
  - "postgresql"
  - "postgresql-database"
  - "silex"
  - "databases"
  - "php"
  - "postgresql"
  - "python"
  - "technology"
---

Don't ask me way, but I need to call to a HTTP server (one [Silex](http://silex.sensiolabs.org/) application) from a PostgreSQL database.

I want to do something like this:

```postgresql
select get('http://localhost:8080?name=Gonzalo')->'hello';
```

PostgreSQL has a datatype for [json](http://www.postgresql.org/docs/9.4/static/functions-json.html). It's really cool and it allows us to connect our HTTP server and our SQL database using same datatype.

PostgreSQL also allows us to create stored procedures using different languages. The default language is [PL/pgSQL](http://www.postgresql.org/docs/9.3/static/plpgsql.html). PL/pgSQL is a simple language where we can embed SQL. But we also can use Python. With Python we can easily create HTTP clients, for example with [urllib2](https://docs.python.org/2/library/urllib2.html). That means that develop our a HTTP client for a PostgreSQL database is pretty straightforward.

```postgresql
CREATE OR REPLACE FUNCTION get(uri character varying)
  RETURNS json AS
$BODY$
import urllib2
 
data = urllib2.urlopen(uri)
 
return data.read()
 
$BODY$
  LANGUAGE plpython2u VOLATILE
  COST 100;
ALTER FUNCTION get(character varying)
  OWNER TO gonzalo;
```

Ok that's a GET client, but we also want a POST client to do something like this:

```postgresql
select post('http://localhost:8080', '{"name": "Gonzalo"}'::json)->'hello';
```

As you can see I want to use application/json instead of application/x-www-form-urlencoded to send request parameters. I wrote about it [here](http://gonzalo123.com/2015/02/02/handling-angularjs-post-requests-with-a-silex-backend/) time ago. So I will create one endpoint within my Silex server to handle my POST requests to:

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use G\AngularPostRequestServiceProvider;
 
$app = new Application(['debug' => true]);
 
$app->register(new AngularPostRequestServiceProvider());
 
$app->post('/', function (Application $app, Request $request) {
    return $app->json(['hello' => $request->get('name')]);
});
 
$app->get('/', function (Application $app, Request $request) {
    return $app->json(['hello' => $request->get('name')]);
});
 
$app->run();
```

And now we only need to create one stored procedure to send POST requests

```postgresql
CREATE OR REPLACE FUNCTION post(
    uri character varying,
    paramenters json)
  RETURNS json AS
$BODY$
import urllib2
 
clen = len(paramenters)
req = urllib2.Request(uri, paramenters, {'Content-Type': 'application/json', 'Content-Length': clen})
f = urllib2.urlopen(req)
return f.read()
 
$BODY$
  LANGUAGE plpython2u VOLATILE
  COST 100;
ALTER FUNCTION post(character varying, json)
  OWNER TO gonzalo;
```

And that's all. At least this simple script is exactly what I need.
