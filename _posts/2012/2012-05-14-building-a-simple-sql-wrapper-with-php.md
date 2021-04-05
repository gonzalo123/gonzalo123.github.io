---
layout: post
title: "Building a simple SQL wrapper with PHP"
date: "2012-05-14"
tags: 
  - "databases"
  - "pdo"
  - "php"
  - "postgresql"
  - "technology"
---

If we don't use an ORM within our projects we need to write SQL statements by hand. I don't mind to write SQL. It's simple and descriptive but sometimes we like to use helpers to avoid write the same code again and again. Today we are going to create a simple library to help use to write simple SQL queries. Let's start:

The idea is to instead of write: 

```php
SELECT * from users where uid=7;
```

write:

```php
$sql->select('users', array('uid' => 7));
```

As we all must know, the best documentation are Unit Test, so here you are the tests of the library:

```php
class SqlTest extends PHPUnit_Framework_TestCase
{
    public function setUp()
    {
        $this->dbh = new Conn('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
        $this->dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $this->dbh->forceRollback();
    }
 
    public function testTransactions()
    {
 
        $sql = new Sql($this->dbh);
        $that = $this;
 
        $this->dbh->transactional(function($dbh) use ($sql, $that) {
            $actual = $sql->insert('users', array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
 
            $actual = $sql->insert('users', array('uid' => 8, 'name' => 'Peter', 'surname' => 'Parker'));
            $that->assertTrue($actual);
 
            $data = $sql->select('users', array('uid' => 8));
            $that->assertEquals('Peter', $data[0]['name']);
            $that->assertEquals('Parker', $data[0]['surname']);
 
            $sql->update('users', array('name' => 'gonzalo'), array('uid' => 7));
 
            $data = $sql->select('users', array('uid' => 7));
            $that->assertEquals('gonzalo', $data[0]['name']);
 
            $data = $sql->delete('users', array('uid' => 7));
 
            $data = $sql->select('users', array('uid' => 7));
            $that->assertTrue(count($data) == 0);
        });
    }
}
```

As you can see we use DI to inject the database connection to our library. Simple isn't it?

Here the whole library: 

```php
class Conn extends PDO
{
    private $forcedRollback = false;
    public function transactional(Closure $func)
    {
        $this->beginTransaction();
        try {
            $func($this);
            $this->forcedRollback ? $this->rollback() : $this->commit();
        } catch (Exception $e) {
            $this->rollback();
            throw $e;
        }
    }
 
    public function forceRollback()
    {
        $this->forcedRollback = true;
    }
}
 
class Sql
{
    /** @var Conn */
    private $dbh;
    function __construct(Conn $dbh)
    {
        $this->dbh = $dbh;
    }
 
    public function select($table, $where)
    {
        $sql         = $this->createSelect($table, $where);
        $whereParams = $this->getWhereParameters($where);
        $stmp = $this->dbh->prepare($sql);
        $stmp->execute($whereParams);
        return $stmp->fetchAll();
    }
 
    public function insert($table, $values)
    {
        $sql = $this->createInsert($table, $values);
        $stmp = $this->dbh->prepare($sql);
        return $stmp->execute($values);
    }
 
    public function update($table, $values, $where)
    {
        $sql = $this->createUpdate($table, $values, $where);
        $whereParams = $this->getWhereParameters($where);
 
        $stmp = $this->dbh->prepare($sql);
        return $stmp->execute(array_merge($values, $whereParams));
    }
 
    public function delete($table, $where)
    {
        $sql         = $this->createDelete($table, $where);
        $whereParams = $this->getWhereParameters($where);
        $stmp = $this->dbh->prepare($sql);
        return $stmp->execute($whereParams);
    }
 
    protected function getWhereParameters($where)
    {
        $whereParams = array();
        foreach ($where as $key => $value) {
            $whereParams[":W_{$key}"] = $value;
        }
        return $whereParams;
    }
 
    protected function createSelect($table, $where)
    {
        return "SELECT * FROM " . $table . $this->createSqlWhere($where);
    }
 
    protected function createUpdate($table, $values, $where)
    {
        $sqlValues = array();
        foreach (array_keys($values) as $key) {
            $sqlValues[] = "{$key} = :{$key}";
        }
        return "UPDATE {$table} SET " . implode(', ', $sqlValues) . $this->createSqlWhere($where);
    }
 
    protected function createInsert($table, $values)
    {
        $sqlValues = array();
        foreach (array_keys($values) as $key) {
            $sqlValues[] = ":{$key}";
        }
        return "INSERT INTO {$table} (" . implode(', ', array_keys($values)) . ") VALUES (" . implode(', ', $sqlValues) . ")";
    }
 
    protected function createDelete($table, $where)
    {
        return "DELETE FROM {$table}" . $this->createSqlWhere($where);
    }
 
    protected function createSqlWhere($where)
    {
        if (count((array) $where) == 0) return null;
 
        $whereSql = array();
        foreach ($where as $key => $value) {
            $whereSql[] = "{$key} = :W_{$key}";
        }
        return ' WHERE ' . implode(' AND ', $whereSql);
    }
}
```

You can see the full code at [github](https://github.com/gonzalo123/sqlWrapper).
