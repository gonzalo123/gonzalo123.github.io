---
layout: post
title: "Foreign Data Wrappers with PostgreSQL and PHP"
date: "2016-02-22"
categories: 

tags: 
  - "php"
  - "postgresql"
  - "technology"
  - "postgresql-database"
  - "silex"
---

PostgreSQL is more than a relational database. It has many cool features. Today we’re going to play with Foreign Data Wrappers (FDW). The idea is crate a virtual table from an external datasource and use it like we use a traditional table.

Let me show you an example. Imagine that we’ve got a REST datasource on port 8888. We’re going to use this Silex application, for example

```php
use Silex\Application;
 
$app = new Application();
 
$app->get('/', function(Application $app) {
 
    return $app->json([
        ['name' => 'Peter', 'surname' => 'Parker'],
        ['name' => 'Clark', 'surname' => 'Kent'],
        ['name' => 'Bruce', 'surname' => 'Wayne'],
    ]);
});
 
$app->run();
```

We want to use this datasource in PostgreSQL, so we need to use a "www foreign data wrapper".

First we create the extension (maybe we need to compile the extension. We can follow the installation instructions [here](https://github.com/cyga/www_fdw))

```postgresql
CREATE EXTENSION www_fdw;
```

Now with the extension we need to create a "server". This server is just a proxy that connects to the real Rest service

```postgresql
CREATE SERVER myRestServer FOREIGN DATA WRAPPER www_fdw OPTIONS (uri 'http://localhost:8888');
```
Now we need to map our user to the server

```postgresql
CREATE USER MAPPING FOR gonzalo SERVER myRestServer;
```

And finally we only need our “Foreign table”

```postgresql
CREATE FOREIGN TABLE myRest (
    name text,
    surname text
) SERVER myRestServer;
```

Now we can perform SQL queries using our Foreign table

```postgresql
SELECT * FROM myRest
```

We must take care with one thing. We can use WHERE clauses but if we run

```postgresql
SELECT * FROM myRest WHERE name='Peter'
```

We’ll that the output is the same than "SELECT * FROM myRest". That’s because if we want to filter something with WHERE clause within Foreign we need to do it in the remote service. WHERE name=‘Peter’ means that our Database will execute the following request: http://localhost:8888?name=Peter

And we need to handle this parameter. For example doing something like that

```php
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application();
 
$app->get('/', function(Application $app, Request $request) {
    $name = $request->get('name');
 
    $data = [
        ['name' => 'Peter', 'surname' => 'Parker'],
        ['name' => 'Clark', 'surname' => 'Kent'],
        ['name' => 'Bruce', 'surname' => 'Wayne'],
    ];
    return $app->json(array_filter($data, function($reg) use($name){
        return $name ? $reg['name'] == $name : true;
    }));
});
 
$app->run();
```
