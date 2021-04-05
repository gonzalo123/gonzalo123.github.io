---
layout: post
title: "Building a simple SQL wrapper with PHP. Part 2."
date: "2012-06-18"
tags: 
  - "pdo"
  - "php"
  - "postgresql"
  - "technology"
---

In one of our last post we built a [simple SQL wrapper with PHP](http://gonzalo123.wordpress.com/2012/05/14/building-a-simple-sql-wrapper-with-php/ "Building a simple SQL wrapper with PHP"). Now we are going to improve it a little bit. We area going to use a class Table instead of the table name. Why? Simple. We want to create triggers. OK we can create triggers directly in the database but sometimes our triggers need to perform operations outside the database, such as call a REST webservice, filesystem's logs or things like that.

```php
<?php
class Storage
{
    static $count = 0;
 
    static function init()
    {
        self::$count = 0;
    }
 
    static function increment()
    {
        self::$count++;
    }
 
    static function decrement()
    {
        self::$count--;
    }
 
    static function get()
    {
        return self::$count;
    }
}
 
class SqlTest extends PHPUnit_Framework_TestCase
{
    public function setUp()
    {
        $this->dbh = new Conn('pgsql:dbname=db;host=localhost', 'gonzalo', 'password');
        $this->dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $this->dbh->forceRollback();
    }
 
    public function testInsertWithPostInsertShowingInsertedValues()
    {
        Storage::init();
        $that = $this;
        $this->dbh->transactional(function($dbh) use ($that) {
            $sql = new Sql($that->dbh);
            $users = new Table('users');
            $users->postInsert(function($values) {Storage::increment();});
 
            $that->assertEquals(0, Storage::get());
            $actual = $sql->insert($users, array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
 
            $that->assertEquals(1, Storage::get());
        });
    }
 
    public function testInsertWithPostInsert()
    {
        Storage::init();
        $that = $this;
        $this->dbh->transactional(function($dbh) use ($that) {
            $sql = new Sql($that->dbh);
            $users = new Table('users');
            $users->postInsert(function() {Storage::increment();});
 
            $that->assertEquals(0, Storage::get());
            $actual = $sql->insert($users, array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
 
            $that->assertEquals(1, Storage::get());
        });
    }
 
    public function testInsertWithPrePostInsert()
    {
        Storage::init();
        $that = $this;
        $this->dbh->transactional(function($dbh) use ($that) {
            $sql = new Sql($that->dbh);
            $users = new Table('users');
            $users->preInsert(function() {Storage::increment();});
            $users->postInsert(function() {Storage::decrement();});
 
            $that->assertEquals(0, Storage::get());
            $actual = $sql->insert($users, array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
 
            $that->assertEquals(0, Storage::get());
        });
    }
 
    public function testUpdateWithPrePostInsert()
    {
        Storage::init();
        $that = $this;
        $this->dbh->transactional(function($dbh) use ($that) {
            $sql = new Sql($that->dbh);
            $users = new Table('users');
            $users->preUpdate(function() {Storage::increment();});
            $users->postUpdate(function() {Storage::increment();});
 
            $that->assertEquals(0, Storage::get());
            $actual = $sql->insert($users, array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
            $that->assertEquals(0, Storage::get());
 
            $data = $sql->select('users', array('uid' => 7));
            $that->assertEquals('Gonzalo', $data[0]['name']);
 
            $actual = $sql->update($users, array('name' => 'gonzalo',), array('uid' => 7));
            $that->assertTrue($actual);
            $that->assertEquals(2, Storage::get());
 
            $data = $sql->select('users', array('uid' => 7));
            $that->assertEquals('gonzalo', $data[0]['name']);
        });
    }
 
    public function testDeleteWithPrePostInsert()
    {
        Storage::init();
        $that = $this;
        $this->dbh->transactional(function($dbh) use ($that) {
            $sql = new Sql($that->dbh);
            $users = new Table('users');
            $users->preDelete(function() {Storage::increment();});
            $users->postDelete(function() {Storage::increment();});
 
            $that->assertEquals(0, Storage::get());
            $actual = $sql->insert($users, array('uid' => 7, 'name' => 'Gonzalo', 'surname' => 'Ayuso'));
            $that->assertTrue($actual);
            $that->assertEquals(0, Storage::get());
 
            $actual = $sql->delete($users, array('uid' => 7));
            $that->assertTrue($actual);
            $that->assertEquals(2, Storage::get());
        });
    }
}
```

And here the whole library: 

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
 
class Table
{
    private $tableName;
 
    function __construct($tableName)
    {
        $this->tableName = $tableName;
    }
 
    private $cbkPostInsert;
    private $cbkPostUpdate;
    private $cbkPostDelete;
    private $cbkPreInsert;
    private $cbkPreUpdate;
    private $cbkPreDelete;
 
    public function getTableName()
    {
        return $this->tableName;
    }
 
    public function postInsert(Closure $func)
    {
        $this->cbkPostInsert = $func;
    }
 
    public function postUpdate(Closure $func)
    {
        $this->cbkPostUpdate = $func;
    }
 
    public function postDelete(Closure $func)
    {
        $this->cbkPostDelete = $func;
    }
 
    public function preInsert(Closure $func)
    {
        $this->cbkPreInsert = $func;
    }
 
    public function preUpdate(Closure $func)
    {
        $this->cbkPreUpdate = $func;
    }
 
    public function preDelete(Closure $func)
    {
        $this->cbkPreDelete = $func;
    }
 
    public function execPostInsert($values)
    {
        $func = $this->cbkPostInsert;
        if ($this->cbkPostInsert instanceof Closure) $func($values);
    }
 
    public function execPostUpdate($values, $where)
    {
        $func = $this->cbkPostUpdate;
        if ($func instanceof Closure) $func($values, $where);
    }
 
    public function execPostDelete($where)
    {
        $func = $this->cbkPostDelete;
        if ($func instanceof Closure) $func($where);
    }
 
    public function execPreInsert($values)
    {
        $func = $this->cbkPreInsert;
        if ($func instanceof Closure) $func($values);
    }
 
    public function execPreUpdate($values)
    {
        $func = $this->cbkPreUpdate;
        if ($func instanceof Closure) $func($values);
    }
 
    public function execPreDelete($where)
    {
        $func = $this->cbkPreDelete;
        if ($func instanceof Closure) $func($where);
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
        $tableName   = ($table instanceof Table) ? $table->getTableName() : $table;
        $sql         = $this->createSelect($tableName, $where);
        $whereParams = $this->getWhereParameters($where);
        $stmp = $this->dbh->prepare($sql);
        $stmp->execute($whereParams);
        return $stmp->fetchAll();
    }
 
    public function insert($table, $values)
    {
        $tableName = ($table instanceof Table) ? $table->getTableName() : $table;
        $sql       = $this->createInsert($tableName, $values);
 
        if ($table instanceof Table) $table->execPreInsert($values);
        $stmp = $this->dbh->prepare($sql);
        $out = $stmp->execute($values);
        if ($table instanceof Table) $table->execPostInsert($values);
        return $out;
    }
 
    public function update($table, $values, $where)
    {
        $tableName   = ($table instanceof Table) ? $table->getTableName() : $table;
        $sql         = $this->createUpdate($tableName, $values, $where);
        $whereParams = $this->getWhereParameters($where);
 
        if ($table instanceof Table) $table->execPreUpdate($values, $where);
        $stmp = $this->dbh->prepare($sql);
        $out = $stmp->execute(array_merge($values, $whereParams));
        if ($table instanceof Table) $table->execPostUpdate($values, $where);
        return $out;
    }
 
    public function delete($table, $where)
    {
        $tableName   = ($table instanceof Table) ? $table->getTableName() : $table;
        $sql         = $this->createDelete($tableName, $where);
        $whereParams = $this->getWhereParameters($where);
 
        if ($table instanceof Table) $table->execPreDelete($where);
        $stmp = $this->dbh->prepare($sql);
        $out = $stmp->execute($whereParams);
        if ($table instanceof Table) $table->execPostDelete($where);
        return $out;
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
