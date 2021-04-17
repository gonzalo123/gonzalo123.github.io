---
layout: post
title: "Performance analysis using bind parameters with PDO and PHP."
date: "2010-10-05"
tags: 
  - "php"
categories: 
  - "php"
  - "postgreSQL"
---

Some months ago a work mate asked me for the differences between using bind variables versus executing the SQL statement directly as a string throughout a PDO connection.

Basically the work-flow of almost all database drivers is the same: Prepare statement, execute and fetch results. We have the following small example with a simple  update

```php
$dbh = new PDO('pgsql:dbname=pg1;host=localhost', 'user', 'pass');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$dbh->beginTransaction();
$stmt = $dbh->prepare("UPDATE test.tbl1 set field1=:F1 where id=1");
$stmt->execute(array('F1' => $field1));
$dbh->commit();
```

And we also can get the same result with the following code:

```php
$dbh = new PDO('pgsql:dbname=pg1;host=localhost', 'user', 'pass');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$dbh->beginTransaction();
$dbh->prepare("UPDATE test.tbl1 set field1='{$field1}' where id=1")->execute();
$dbh->commit();
```

What’s the best one? Both method work properly. The difference is how databases manage the operation internally. When we prepare a statement we are compiling the string into the database within the current connection. After that we can execute the statement and if our statement is ready to receive parameters we can bind those parameters with PHP values or variables. With this idea in mind we can guess that if we need to perform several executions of the same prepared statement is better to use bind parameters instead of compile and execute directly the string. b I have created a simple benchmark to show it.

```php
<?php
error_reporting(-1);
function microtime_float()
{
   list($usec, $sec) = explode(" ", microtime());
   return ((float)$usec + (float)$sec);
}
 
$dbh = new PDO('pgsql:dbname=pg1;host=localhost', 'user', 'pass');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$time_start = microtime_float();
$dbh->beginTransaction();
$field1 = uniqid();
$stmt = $dbh->prepare("UPDATE test.tbl1 set field1=:F1 where id=1");
$stmt->execute(array('F1' => $field1));
$dbh->commit();
$time_end = microtime_float();
$time = round($time_end - $time_start, 4);
echo "<p>Single UPDATE with bind parameters: $time seconds</p>";
 
$time_start = microtime_float();
$dbh->beginTransaction();
$field1 = uniqid();
$stmt = $dbh->prepare("UPDATE test.tbl1 set field1='{$field1}' where id=1");
$stmt->execute();
$dbh->commit();
$time_end = microtime_float();
$time = round($time_end - $time_start, 4);
echo "<p>Single UPDATE without bind parameters: $time seconds</p>";
 
$time_start = microtime_float();
$dbh->beginTransaction();
$stmt = $dbh->prepare('UPDATE test.tbl1 set field1=:F1 where id=1');
foreach (range(1, 5000, 1) as $i) {
   $field1 = $i;
   $stmt->execute(array('F1' => $field1));
}
$dbh->commit();
$time_end = microtime_float();
$time = round($time_end - $time_start, 4);
echo "<p>Multiple UPDATE with bind parameters: $time seconds</p>";
 
$time_start = microtime_float();
$dbh->beginTransaction();
foreach (range(1, 5000, 1) as $i) {
    $stmt = $dbh->prepare("UPDATE test.tbl1 set field1='{$field1}' where id=1");
   $field1 = $i;
   $stmt->execute();
}
$dbh->commit();
$time_end = microtime_float();
$time = round($time_end - $time_start, 4);
echo "<p>Multiple UPDATE without bind parameters: $time seconds</p>";
```

The output of this benchmark is the following one:

```
Single UPDATE with bind parameters: 0.2623 seconds
Single UPDATE without bind parameters: 0.0195 seconds
Multiple UPDATE with bind parameters: 4.1123 seconds
Multiple UPDATE without bind parameters: 8.1732 seconds
```

As we can see in the output of the benchmark a single update is slightly faster without bind any parameter but if we need to execute the same update with different parameters the bind parameters way is significantly faster than parse+execute again and again.

There’s another benefit of using bind parameters. Databases normally have internally a cache system for our prepared statement. In theory they reuse the statements. The main problem for us (in our PHP world) is that normally we create a new connection to the database at the beginning of the script (or lazy connection) and close it at the end. We don’t have natively a connection pooling like J2EE. So I’m not 100% sure if using bind parameters helps to the database to reuse statements.

There's another issue we must take into account. Without prepared statements the multiple update example may throw a database exception (depend on our database configuration). Too many open cursors in active transaction. That error appear because for our database each update is a different statement (5000 in the example) and with the another method there's only one.
