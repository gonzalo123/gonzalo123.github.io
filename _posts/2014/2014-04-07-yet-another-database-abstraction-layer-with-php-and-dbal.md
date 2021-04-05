---
layout: post
title: "Yet another Database Abstraction layer with PHP and DBAL"
date: "2014-04-07"
tags: 
  - "dbal"
  - "pdo"
  - "sql"
  - "databases"
  - "dbal"
  - "php"
  - "technology"
---

I'm not a big fan of ORMs. I feel very confortable working with raw SQLs and because of that I normally use [DBAL](http://www.doctrine-project.org/projects/dbal.html) (or [PDO](http://php.net/pdo) in old projects). I've got one small library to handle my dayly operations with databases and today I've written this library

First of all imagine one DBAL connection. I'm using a sqlite in-memomy database in this example but we can use any database supported by DBAL (aka "almost all"):

```php
use Doctrine\DBAL\DriverManager;
 
$conn = DriverManager::getConnection([
    'driver' => 'pdo_sqlite',
    'memory' => true
]);
```

We can also create one DBAL connection from a PDO connection. It's usefull to use DBAL within legacy applications instead of creating a new connection (remember that DBAL works over PDO)

```php
use Doctrine\DBAL\DriverManager;
 
$conn = DriverManager::getConnection(['pdo' => new PDO('sqlite::memory:')]);
```

Now we set up the database for the example 

```php
$conn->exec("CREATE TABLE users (
            userid VARCHAR PRIMARY KEY  NOT NULL ,
            password VARCHAR NOT NULL ,
            name VARCHAR,
            surname VARCHAR
            );");
$conn->exec("INSERT INTO users VALUES('user','pass','Name','Surname');");
$conn->exec("INSERT INTO users VALUES('user2','pass2','Name2','Surname2');")
```

Our table "users" has two records. Now we can start to use our library.

First we create a new instance of our library: 

```php
use G\Db;
 
$db = new Db($conn);
```

Now a simple query from a string: 

```php
$data = $db->select("select * from users");
```

Sometimes I'm lazy and I don't want to write the whole SQL string and I want to perform a select \* from table: 

```php
use G\Sql;
$data = $db->select(SQL::createFromTable("users"));
```

Probably we need to filter our Select statement with a WHERE clause: 

```php
$data = $db->select(SQL::createFromTable("users", ['userid' => 'user2']));
```

And now something very intersting (at least for me). I want to iterate over the recordset and maybe change it. Of course I can use "foreach" over $data and do whatever I need, but I preffer to use the following syntax: 

```php
$data = $db->select(SQL::createFromTable("users"), function (&$row) {
    $row['name'] = strtoupper($row['name']);
});
```

For me it's more readable. I iterate over the recordset and change the row 'name' to uppercase. Here you can see what is doing my "select" function:

```php
/**
* @param Sql|string $sql
* @param \Closure $callback
* @return array
*/
public function select($sql, \Closure $callback = null)
{
    if ($sql instanceof Sql) {
        $sqlString = $sql->getString();
        $parameters = $sql->getParameters();
        $types = $sql->getTypes();
    } else {
        $sqlString = $sql;
        $parameters = [];
        $types = [];
    }
 
    $statement = $this->conn->executeQuery($sqlString, $parameters, $types);
    $data = $statement->fetchAll();
    if (!is_null($callback) && count($data) > 0) {
        $out = [];
        foreach ($data as $row) {
            if (call_user_func_array($callback, [&$row]) !== false) {
                $out[] = $row;
            }
        }
        $data = $out;
   }
 
   return $data;
}
```

And finally transactions (I normally never use autocommit and I like to handle transactions by my own)

```php
$db->transactional(function (Db $db) {
    $userId = 'temporal';
 
    $db->insert('users', [
        'USERID'   => $userId,
        'PASSWORD' => uniqid(),
        'NAME'     => 'name3',
        'SURNAME'  => 'name3'
    ]);
 
    $db->update('users', ['NAME' => 'updatedName'], ['USERID' => $userId]);
    $db->delete('users', ['USERID' => $userId]);
});
```

The "transactional" function it's very simmilar than DBAL's transactional function

```php
public function transactional(\Closure $callback)
{
    $out = null;
    $this->conn->beginTransaction();
    try {
        $out = $callback($this);
        $this->conn->commit();
    } catch (\Exception $e) {
        $this->conn->rollback();
        throw $e;
    }
 
    return $out;
}
```

I change a little bit because I like to return a value within the closure and allow to do things like that: \

```php
$status = $db->transactional(function (Db $db) {
    $userId = 'temporal';
 
    $db->insert('users', [
        'USERID'   => $userId,
        'PASSWORD' => uniqid(),
        'NAME'     => 'name3',
        'SURNAME'  => 'name3'
    ]);
 
    $db->update('users', ['NAME' => 'updatedName'], ['USERID' => $userId]);
    $db->delete('users', ['USERID' => $userId]);
 
    return "OK"
});
```

The other functions (insert, update, delete) only bypass the calls to DBAL's functions: 

```php
private $conn;
 
public function __construct(Doctrine\DBAL\Connection $conn)
{
    $this->conn = $conn;
}
 
public function insert($tableName, array $values = [], array $types = [])
{
    $this->conn->insert($tableName, $values, $types);
}
 
public function delete($tableName, array $where = [], array $types = [])
{
    $this->conn->delete($tableName, $where, $types);
}
 
public function update($tableName, array $data, array $where = [], array $types = [])
{
    $this->conn->update($tableName, $data, $where, $types);
}
```

And that's all. You can use the library with [composer](https://packagist.org/packages/gonzalo123/db) and download at [github](https://github.com/gonzalo123/Db).

