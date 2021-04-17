---
layout: post
title: "Playing with PHP, CouchDB and ORM"
date: "2010-03-16"
tags: 
  - "php"
  - "couchDB"
categories: 
  - "php"
  - "couchDB"
---

I'm playing with [PHP and CouchDB](http://gonzalo123.wordpress.com/2010/03/15/php-and-couchdb/). After writing a [class](http://code.google.com/p/nov-couchdb/) to connect PHP and CouchDB I've done a variation of the interface to use CouchDB with PHP in a similar way to ORM. Yes. I know [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping) means Object Relational Mapping and [CouchDB](http://couchdb.apache.org/) isn't a Relational Database but have a look to the following interface:

Imagine you have a CouchDb at localhost:5987 and you have a database called _**users**_.

Now let's build a class called CDB \Users.

```php
class CDB_Users extends Nov_CouchDb_Orm {
    static $_db = 'users';
}
```

It's easy to automate the creation of CDB \Users with a script

CDB \Users extends Nov \CouchDb \Orm

```php
class Nov_CouchDb_Orm
{
    static protected $_db;
 
    /**
     * @param string $key
     * @return Nov_CouchDb
     */
    static public function connect($key)
    {
        $class = get_called_class();
        extract(Nov_CouchDb_Conf::get($key));
 
        $couchDb = new Nov_CouchDb($host, $port, $protocol, $user, $password);
        return $couchDb->db($class::$_db);
    }
}
```

A configuration class:

```php
class Nov_CouchDb_Conf
{
    const CDB1 = 'CDB1';
 
    private static $_conf = array(
        self::CDB1 => array(
            'protocol' => 'http',
            'host'     => 'localhost',
            'port'     => 5984,
            'user'     => null,
            'password' => null
            )
        );
 
    static function get($key)
    {
        return self::$_conf[$key];
    }
}
```

Putting all together with the class Nov \CouchDb we can use:

```php
CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->insert('gonzalo', array('password' => 'g1'));
$data = CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->select('gonzalo')->asArray();
print_r($data);
 
CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->update('gonzalo', array('password' => 'g2'));
$data = CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->select('gonzalo')->asArray();
print_r($data);
 
CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->delete('gonzalo');
```

What do you think? That's not exactly a ORM. I'm not sure if I will use that interface instead the classic one.

```php
$data = CDB_Users::connect(Nov_CouchDb_Conf::CDB1)->select('gonzalo')->asArray();
// versus
$couchDb = new Nov_CouchDb('localhost', 5984);
$data = $couchDb->db('users')->select('gonzalo')->asArray()
```

[Here](http://code.google.com/p/nov-couchdb/) is the full source code with the examples.
