---
layout: post
title: "Performance analysis fetching data with PDO and PHP."
date: "2011-03-28"
tags: 
  - "pdo"
  - "php"
  - "postgresql"
  - "pdo"
  - "performance"
---

Fetching data from databases is a common operation in our work as developers. There are many drivers (normally I use PDO), but the usage of all of them are similar and switch from one to another is not difficult (they almost share the same interface). In this post I will focus on fetching data. Basically we've got two functions: [fetch](http://www.php.net/manual/en/pdostatement.fetch.php) and [fetchAll](http://php.net/manual/en/pdostatement.fetchall.php). I've created two examples. One with fetch and another one with fetchAll:

```php
// Example with fetch
error_reporting(-1);
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$dbh = new PDO('pgsql:dbname=mydb;host=localhost', 'username', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$stmt = $dbh->prepare('SELECT * FROM tableName limit 10000');
$stmt->execute();
 
$i=0;
while ($row = $stmt->fetch()) {
    $i++;
}
echo '
<h1>fetch()</h1>
';
echo '
<strong>{$i} </strong>
 
';
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));

```

```php
// Example with fetchAll
error_reporting(-1);
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$dbh = new PDO('pgsql:dbname=mydb;host=localhost', 'username', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$stmt = $dbh->prepare('SELECT * FROM tableName limit 10000');
$stmt->execute();
 
$i=0;
$data = $stmt->fetchAll();
foreach ($data as $row) {
    $i++;
}
 
echo '
<h1>fetchAll()</h1>
';
echo '
<strong>{$i}</strong>
 
';
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));
```

if we execute the test we obtain:

**fetchAll**: \[memory\] => 31.305999755859
**fetch**: \[memory\] => 0.002532958984375

OK. It's obvious. If we approach to the data extraction with fetchAll method we will use more memory. That's because we're mapping the whole recorded to a variable ($data) at once. With the fetch loop we are mapping only on row per iteration. By the way if we change the fetch loop to:

```php
$data = array();
while ($row = $stmt->fetch()) {
    $i++;
    $data[] = $row;
}
```

We will use almost the same amount of memory than the fetchAll method \[memory\] => 31.267543792725

**Conclusion**: 
Is it better fetch than fetchAll? The answer is simple: No. We only need to take care what are we doing and use the best solution that fix to our need. If we're handling small recordset, they're similar, but if we work with big ones we need to realize that the memory usage we are using changes drastically if we use one method or another.
