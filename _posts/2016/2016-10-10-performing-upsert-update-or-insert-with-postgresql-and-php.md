---
layout: post
title: "Performing UPSERT (Update or Insert) with PostgreSQL and PHP"
date: "2016-10-10"
categories: 

tags: 
  - "dbal"
  - "pdo"
  - "php"
  - "postgresql"
  - "technology"
  - "sql"
---

That's a typical situation. Imagine you've got one table

```postgresql
CREATE TABLE PUBLIC.TBUPSERTEXAMPLE
(
  KEY1 CHARACTER VARYING(10) NOT NULL,
  KEY2 CHARACTER VARYING(14) NOT NULL,
  KEY3 CHARACTER VARYING(14) NOT NULL,
  KEY4 CHARACTER VARYING(14) NOT NULL,
 
  VALUE1 CHARACTER VARYING(20),
  VALUE2 CHARACTER VARYING(20) NOT NULL,
  VALUE3 CHARACTER VARYING(100),
  VALUE4 CHARACTER VARYING(400),
  VALUE5 CHARACTER VARYING(20),
 
  CONSTRAINT TBUPSERTEXAMPLE_PKEY PRIMARY KEY (KEY1, KEY2, KEY3, KEY4)
)
```

And you need to update one record. You can perform a simple UPDATE statement but what happens the first time?

You cannot update the record basically because the record doesn't exists. You need to create an INSERT statement instead. We can do it following different ways. You can create first a SELECT statement and, if the record exists, perform an UPDATE. If it doesn't exists you perform an INSERT. We also can perform an UPDATE and see how many records are affected. If no records are affected then we perform an INSERT. Finally we can perform one INSERT and it it throws an error then perform an UPDATE.

All of these techniques works in one way or another but [PostgreSQL](https://wiki.postgresql.org/wiki/UPSERT) gives us one cool way of doing this operation with one SQL sentence. We can use CTE (Common Table Expression) and execute something like this:

```postgresql
WITH upsert AS (
    UPDATE PUBLIC.TBUPSERTEXAMPLE
    SET
        VALUE1 = :VALUE1,
        VALUE2 = :VALUE2,
        VALUE3 = :VALUE3,
        VALUE4 = :VALUE4,
        VALUE5 = :VALUE5
    WHERE
        KEY1 = :KEY1 AND
        KEY2 = :KEY2 AND
        KEY3 = :KEY3 AND
        KEY4 = :KEY4
    RETURNING *
)
INSERT INTO PUBLIC.TBUPSERTEXAMPLE(KEY1, KEY2, KEY3, KEY4, VALUE1, VALUE2, VALUE3, VALUE4, VALUE5)
SELECT
    :KEY1, :KEY2, :KEY3, :KEY4, :VALUE1, :VALUE2, :VALUE3, :VALUE4, :VALUE5
WHERE
    NOT EXISTS (SELECT 1 FROM upsert);
```

Since PostgreSQL 9.5 we also can do another technique to do this UPSERT operations. We can do something like this: 

```postgresql
INSERT INTO PUBLIC.TBUPSERTEXAMPLE (key1, key2, key3, key4, value1, value2, value3, value4, value5)
  VALUES ('key2', 'key2', 'key3', 'key4', 'value1',  'value2',  'value3',  'value4',  'value5')
ON CONFLICT (key1, key2, key3, key4)
DO UPDATE SET
  value1 = 'value1', 
  value2 = 'value2', 
  value3 = 'value3', 
  value4 = 'value4', 
  value5 = 'value5'
WHERE
  TBUPSERTEXAMPLE.key1 = 'key2' AND
  TBUPSERTEXAMPLE.key2 = 'key2' AND
  TBUPSERTEXAMPLE.key3 = 'key3' AND
  TBUPSERTEXAMPLE.key4 = 'key4';
```

To help me writing this sentence I've created a simple PHP wrapper:

Here one example using PDO

```php
use G\SqlUtils\Upsert;
 
$conn = new PDO('pgsql:dbname=gonzalo;host=localhost', 'username', 'password');
$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$conn->beginTransaction();
try {
    Upsert::createFromPDO($conn)->exec('PUBLIC.TBUPSERTEXAMPLE', [
        'KEY1' => 'key1',
        'KEY2' => 'key2',
        'KEY3' => 'key3',
        'KEY4' => 'key4',
    ], [
        'VALUE1' => 'value1',
        'VALUE2' => 'value2',
        'VALUE3' => 'value3',
        'VALUE4' => 'value4',
        'VALUE5' => 'value5',
    ]);
    $conn->commit();
} catch (Exception $e) {
    $conn->rollback();
    throw $e;
}
```

And another one using DBAL 

```php
use Doctrine\DBAL\DriverManager;
use G\SqlUtils\Upsert;
 
$connectionParams = [
    'dbname'   => 'gonzalo',
    'user'     => 'username',
    'password' => 'password',
    'host'     => 'localhost',
    'driver'   => 'pdo_pgsql',
];
 
$dbh = DriverManager::getConnection($connectionParams);
$dbh->transactional(function ($conn) {
    Upsert::createFromDBAL($conn)->exec('PUBLIC.TBUPSERTEXAMPLE', [
        'KEY1' => 'key1',
        'KEY2' => 'key2',
        'KEY3' => 'key3',
        'KEY4' => 'key4',
    ], [
        'VALUE1' => 'value1',
        'VALUE2' => 'value2',
        'VALUE3' => 'value3',
        'VALUE4' => null,
        'VALUE5' => 'value5',
    ]);
});
```

And that's all. Library is available in my [github](https://github.com/gonzalo123/sqlutils) and it's also at [packagist](https://packagist.org/packages/gonzalo123/sqlutils).
