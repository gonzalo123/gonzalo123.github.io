---
layout: post
title: "How to protect from SQL Injection with PHP"
date: "2012-02-06"
tags: 
  - "databases"
  - "pdo"
  - "php"
  - "technology"
  - "pdo"
  - "postgresql"
  - "sqlinjection"
---

Security is a part of our work as developers. We need to ensure our applications against malicious attacks. SQL Injection is one of the most common possible attacks. Basically SQL Injection is one kind of attack that happens when someone injects SQL statements in our application. You can find a lot of info about [SQL Injection](http://en.wikipedia.org/wiki/SQL_injection) attack. Basically you need to follow the security golden rule

> Filter input Escape output

If you work with PHP problably you work with PDO Database abstraction layer. Let’s prepare our database for the examples (I work with PostgreSQL):

```postgresql
CREATE TABLE users
(
  uid integer NOT NULL,
  name character varying(50),
  surname character varying(50),
  CONSTRAINT pk_users PRIMARY KEY (uid)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE users OWNER TO gonzalo;
 
INSERT INTO users(uid, name, surname) VALUES (0, 'Gonzalo', 'Ayuso');
INSERT INTO users(uid, name, surname) VALUES (1, 'Hans', 'Solo');
INSERT INTO users(uid, name, surname) VALUES (2, 'Luke', 'Skywalker');
```

OK our database is ready. Now let create a simple query 

```php
$dbh = new PDO('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$stmt = $dbh->prepare('select uid, name, surname from users where uid=:ID');
$stmt->execute(array('ID' => 0));
$data = $stmt->fetchAll();
```

It works. We are using bind parameters, so we need to prepare one statement, execute and fetch the recordset. The use of prepared statements is strongly recommended. We can also use query() function:

```php
$dbh = new PDO('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$uid = 0;
$stmt = $dbh->query("select uid, name, surname from users where uid={$uid}");
$data = $stmt->fetchAll();
```

But what happens if $id came from the request and it's not propertly escaped 

```php
$dbh = new PDO('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$uid = "0; drop table users;";
$stmt = $dbh->query("select uid, name, surname from users where uid={$uid}");
$data = $stmt->fetchAll();
```

basically nothing: SQLSTATE\[42601\]: Syntax error. That's because is not allowed to use two prepared statements in a single statement.

If we use an insert:

```php
$dbh = new PDO('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$uid     = 20;
$name    = 'Gonzalo';
$surname = "Ayuso'); drop table users; select 1 where ('1' = '1";
 
$count = $dbh->exec("INSERT INTO users(uid, name, surname) VALUES ({$uid}, '{$name}', '{$surname}')");
```

Now we have a problem. Our user table will be deleted. Why? That's because of the user we are using to connect to the database. It's important especially at production servers. It's very important not to use a database superuser in production. Superusers are very comfortable in our development servers, because you don't need to grant privileges to every tables but if you forget this issue in production you could have Sql-Injection problems. The solution is very simple:

```php
GRANT SELECT, UPDATE, INSERT, DELETE ON TABLE users TO gonzalo2;
```

And now 

```php
$dbh = new PDO('pgsql:dbname=db;host=localhost', 'gonzalo2', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$uid     = 20;
$name    = 'Gonzalo';
$surname = "Ayuso'); drop table users; select 1 where ('1' = '1";
 
$count = $dbh->exec("INSERT INTO users(uid, name, surname) VALUES ({$uid}, '{$name}', '{$surname}')");
```

Now we are safe, at least with this possible attack. Sumarizing:

- Filter input
- Escape output
- **Take care about the database users**. Don't use one user that it allowed to perform "not-allowed" operations within our application. It sounds like a pun but is important.
