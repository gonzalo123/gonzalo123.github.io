---
layout: post
title: "Building an ORM with PHP."
date: "2010-09-13"
tags: 
  - "php"
categories: 
  - "php"
---

First of all I must clear one thing. If you are looking for a full ORM you must take a look to [Doctrine](http://www.doctrine-project.org/) or maybe to the new version, still not stable, [Doctrine2](http://www.doctrine-project.org/blog/doctrine2-preview-release). Doctrine is probably the most advanced PHP project that we can find. Having said that let's start with **Nov/****Orm.** What’s the motivation for me to build this ORM? The answer is a bit ambiguous. I like SQL. It allows us to speak with the database in a very easy way. When I want to connect with the database with PHP I like [PDO](http://www.php.net/manual/en/book.pdo.php). It’s a great extension. But my problem is the following one: I need to write SQL and I need to know the names of the tables and the names of the fields and write them within my SQL string again and again. So the idea I figured out was to create a set of classes based on my tables, in a similar way than traditional ORMs to help me to autocomplete the fields and table names. PHP’s IDEs help us with the autocomplete of PHP but not with SQL. You must change the perspective or even the software. If my tables are mapped with PHP objects in a particular way I can get two objectives at the same time.

## First basic example:

Let’s show a usage example. Imagine we have a table called _tbl1_ in the test schema. And we want to perform a simple select:

```postgresql
CREATE TABLE test.tbl1
(
    id integer NOT NULL,
    field1 character varying,
    field3 timestamp without time zone
)
WITH (
OIDS=FALSE
);
```

```postgresql
SELECT * FROM test.tbl1
```

We can easily use PDO with this SQL and fetch the results. As I told before we have created a class for our table (later we’ll see the code). So now we can execute:

```php
$all = test\tbl1::factory($db)->select()->exec();
```

If we have properly set the IDE when we type test\\ an autocompletion pop up will appear.

## Mapped clases:

Each database table will be mapped into two PHP classes. One for the table indeed and another for the recordset associated to the table.  Imagine our table test.tbl1 has three fiels: id, field1 and field2. Our first class will be allocated into _Orm__\\test_ namespace. "Orm" because all classes will be there and "test" because the schema name is test.

```php
namespace Orm\test;
use \Nov;
class tbl1 extends Nov\Db\Orm\Table
{
    protected $_schema = 'test';
    protected $_object = "tbl1";
 
    const ID = "id";
    const FIELD1 = "field1";
    const FIELD3 = "field3";
 
    protected $_conf = array(
        self::ID => array(Nov\Types::STR),
        self::FIELD1 => array(Nov\Types::STR),
        self::FIELD3 => array(Nov\Types::STR),
    );
}
```

The second class is for the recordset. Why this class? The answer is because PHP’s lack of strong typed variables. It’s very typical working with databases to iterate over a recordset.

```php
$all = test\tbl1::factory($db)->select()->exec();
foreach ($all as $reg){
 ....
}
```

But $reg is a class and is difficult to hint to the IDE what class is it. Here you can find different ways to solve the [problem](http://gonzalo123.wordpress.com/2010/06/20/phpdoc-code-completion-and-iterations-with-php/). But as well as we need to create the first mapped class we also can create a second helper one (Those classes will be created with a script not by hand)

```php
$all = test\tbl1::factory($db)->select()->exec();
foreach ($all as $reg){
 $ar = new test\tbl1_Record($reg);
 echo $ar->id() . $ar->field1()
}
```

That’s the second mapped class:

```php
class tbl1_Record
{
    /**
    * @param \Nov\Orm\Record $recordset
    * @return \Orm\test\tbl1_Record
    */
    static function factory($recordset)
    {
        return new tbl1_Record($recordset);
    }
 
    private $_recordset;
    function __construct($recordset)
    {
        $this->_isObject = $recordset instanceof Nov\Db\Orm\Record;
        $this->_recordset = $recordset;
    }
 
    function id()
    {
        return $this->_isObject ? $this->_recordset->{tbl1::ID} : $this->_recordset[tbl1::ID];
    }
 
    function field1()
    {
        return $this->_isObject ? $this->_recordset->{tbl1::FIELD1} : $this->_recordset[tbl1::FIELD1];
    }
 
    function field3()
    {
        return $this->_isObject ? $this->_recordset->{tbl1::FIELD3} : $this->_recordset[tbl1::FIELD3];
    }
}
```

And basically that’s all now with the library **Nov\\Orm** we can create script like this:

```php
// Setting up the library
require_once("Nov/Loader.php");
Nov\Loader::init();
 
// setting up namespaces
use Nov\Db\Orm\Instance;
use Orm\test;
 
echo "<pre>";
// New connection to dabase (lazy connection)
$db = Nov\Db::factory(NovConf::PG1);
$db->beginTransaction();
 
$values = array('id' => 'max(' . test\tbl1::ID . ')');
// Select to a table
$id = test\tbl1::factory($db)->select($values)->exec(Nov\Db::FETCH_ONE);
$id++;
$values = array(
    test\tbl1::ID => $id,
    test\tbl1::FIELD1 => "user_{$id}"
);
// Insert
test\tbl1::factory($db)->insert($values)->exec();
 
// Update
test\tbl1::factory($db)->update(array(test\tbl1::FIELD1 => 'xxx'))->where(array(test\tbl1::ID => $id))->exec();
$db->commit();
 
// another selec
$all = test\tbl1::factory($db)->select()->exec();
 
// Iterating over the recorset
foreach ($all as $reg) {
    $ar = new test\tbl1_Record($reg);
    echo $ar->id();
    echo "::";
    echo $ar->field1();
    echo "\n";
}
 
// Select to a view
$all = test\vw1::factory($db)->select()->exec();
foreach ($all as $reg) {
    $ar = new test\vw1_Record($reg);
    echo $ar->id();
    echo "::";
    echo $ar->field1();
    echo "\n";
}
echo "</pre>";
```

## Triggers:

There’s also an extra feature. [Triggers](http://en.wikipedia.org/wiki/Database_trigger) over ORM. If we create by hand a class like this we’ll have the traditional triggers post/pre Insert, update and delete:

```php
namespace Orm\test\tbl1;
use Nov\Db\Orm\Instance\Interfaces;
 
use Orm\test;
use Nov;
 
class Triggers extends Nov\Db\Orm\Trigger
{
    function triggerPostUpdate($where, $values)
    {
        echo __CLASS__ . "::" . __FUNCTION__ . "\n";
        var_export($where);
        echo "\n";
        var_export($values);
        echo "\n";
        test\tbl1::factory($this->getDb())
            ->delete()
            ->where($where)
            ->exec();
    }
 
    function triggerPostInsert($values)
    {
        var_export($values);
        echo "\n";
        echo __CLASS__ . "::" . __FUNCTION__ . "\n";
    }
 
    function triggerPostDelete($where)
    {
        var_export($where);
        echo "\n";
        echo __CLASS__ . "::" . __FUNCTION__ . "\n";
    }
}
```

## Scaffolding:

There is also a simple scaffolding to create the mapped classes.

```php
require_once("Nov/Loader.php");
Nov\Loader::init();
 
use Nov\Db\Orm;
$scaffold = new Orm\Scaffold(Nov\Db::factory(NovConf::PG1), 'test');
$scaffold->buildAll(NovBASEPATH.'/Orm');
```

## Summary:

Nov/Orm has those features:

- Works over PDO extension. That’s means may be compatible with all databases compatible with PDO. The problem is to create the scaffold script to create the mapped classes in the selected database. The library now has only PostgreSql support. But with a little knowledge of the data dictionary of the database will be easy to implement Oracle and Mysql support.
- Triggers over ORM
- The autocompletion over IDE has been taken into account. The main purpose of the library is help the developer to write scripts.
- Iterator helper
- The main problem of this library is obvious. We need to create the Mapped classes if we want to use then. If we alter or add a new table we need to execute the scaffolding script.

The library with examples is available [here](http://code.google.com/p/nov-framework/). The examples are under document\_root/tests/orm/
