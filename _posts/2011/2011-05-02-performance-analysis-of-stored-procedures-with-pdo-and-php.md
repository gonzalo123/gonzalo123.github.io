---
layout: post
title: "Performance analysis of Stored Procedures with PDO and PHP"
date: "2011-05-02"
tags: 
  - "databases"
  - "pdo"
  - "postgresql"
  - "technology"
  - "pdo"
  - "performance"
  - "php"
  - "postgresql"
---

Last week I had an interesting conversation on [twitter](http://twitter.com/#!/gonzalo123) about the usage of stored procedures in databases. Someone told stored procedure are evil. I'm not agree with that. Stored procedures are a great place to store business logic. In this post I'm going to test the performance of a small piece of code using stored procedures and using only PHP code.

**Without** stored procedures 

```php
// Without stored procedures
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$dsn = 'pgsql:host=localhost;dbname=gonzalo;port=5432';
$user = 'user';
$password = 'password';
$conn = new PDO($dsn, $user, $password);
$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$conn->beginTransaction();
$stmt = $conn->prepare('delete from web.tbltest');
$stmt->execute();
 
$stmt = $conn->prepare('INSERT INTO web.tbltest (field1) values (?)');
foreach (range(0,1000) as $i) {
    $stmt->execute(array($i));
}
$conn->commit();
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));

```

**With** stored procedures:

```php
// With stored procedures:
/*
CREATE OR REPLACE FUNCTION web.method1()
  RETURNS numeric AS
$BODY$
BEGIN
   DELETE FROM web.tbltest;
   FOR i IN 0..1000 LOOP
     INSERT INTO web.tbltest (field1) values (i);
   END LOOP;
   RETURN 1;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
*/
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$dsn = 'pgsql:host=localhost;dbname=gonzalo;port=5432';
$user = 'user';
$password = 'password';
$conn = new PDO($dsn, $user, $password);
$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$conn->beginTransaction();
$stmt = $conn->prepare('SELECT web.method1()');
$stmt->execute();
$stmt->setFetchMode(PDO::FETCH_ASSOC);
$out = $stmt->fetchAll();
$conn->commit();
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));
```

<table><tbody><tr><td><b>without stored procedures</b></td><td><b>with stored procedures</b></td></tr><tr><td>memory: 0.0023880004882812<br>seconds: 0.31109309196472</td><td>memory: 0.0020713806152344<br>seconds: 0.065021991729736</td></tr></tbody></table>

So my conclusion: Stored procedures are not evil. The performance is really good. I know maybe it can be a bit mess if we place business logic within database and outside database at the same time, but with a good design and architecture this problem is easy to solve. What do you think?
