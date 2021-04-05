---
layout: post
title: "Database connection pooling with PHP and React (node.php)"
date: "2012-05-21"
tags: 
  - "databases"
  - "pdo"
  - "php"
  - "postgresql"
  - "technology"
  - "language-php"
  - "node-php"
  - "programming"
  - "react"
---

Last saturday I meet a new hype: “[React](http://nodephp.org/)” also known as “node.php”. Basically it’s the same idea than node.js but built with PHP instead of javascript. Twitter was on fire with this new library (at least my [timeline](http://twitter.com/#!/gonzalo123)). The next sunday was a rainy day and because of that I spent the afternoon hacking a little bit with this new library. Basically I want to create a database connection pooling. It’s one of the things that I miss in PHP. I wrote a post here some time ago with this idea with one exotic experiment building one [connection pooling using gearman](http://gonzalo123.wordpress.com/2010/11/01/database-connection-pooling-with-php-and-gearman/). Today the idea is the same but now with React. Let’s start

First of all we install React. It's a simple process using [composer](http://getcomposer.org/). 

```commandline
% echo '{ "require": { "react/react": "dev-master" } }' > composer.json
% composer install
```

Now we can start with our experiment. Imagine a simple query to PostgreSql using PDO: 

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

```php
$dbh = new PDO('pgsql:dbname=demo;host=vmserver', 'gonzalo', 'password');
$sql = "SELECT * FROM USERS";
$stmt = $dbh->prepare($sql);
$stmt->execute();
$data = $stmt->fetchAll();
print_r($data);
```

Now we are going to use the same interface but instead of using PDO we will use one server with React: 

```php
include "CPool.php";
define('NODEPHP', '127.0.0.1:1337');
 
$dbh = new CPool();
$sql = "SELECT * FROM USERS";
$stmt = $dbh->prepare($sql);
$stmt->execute();
$data = $stmt->fetchAll();
$stmt->closeCursor();
print_r($data);
```

Our CPool library: 
```php
class CPoolStatement
{
    private $stmt;
    function __construct($sql=null)
    {
        if (!is_null($sql)) {
            $url = "http://" . NODEPHP . "?" . http_build_query(array(
                    'action' => 'prepare',
                    'sql'    => $sql
                 ));
            $this->stmt = file_get_contents($url);
        }
    }
 
    public function getId()
    {
        return $this->stmt;
    }
 
    public function setId($id)
    {
        $this->stmt = $id;
    }
 
    public function execute($values=array())
    {
        $url = "http://" . NODEPHP . "?" . http_build_query(array(
                'action' => 'execute',
                'smtId'  => $this->stmt,
                'values' => $values
             ));
        $this->stmt = file_get_contents($url);
    }
 
    public function fetchAll()
    {
        $url = "http://" . NODEPHP . "?" . http_build_query(array(
                'action' => 'fetchAll',
                'smtId'  => $this->stmt
             ));
        return (file_get_contents($url));
    }
 
    public function closeCursor()
    {
        $url = "http://" . NODEPHP . "?" . http_build_query(array(
                'action' => 'closeCursor',
                'smtId'  => $this->stmt
             ));
        return (file_get_contents($url));
    }
}
 
class CPool
{
    function prepare($sql)
    {
        return new CPoolStatement($sql);
    }
}
```

We also can create one script that creates one statement 

```php
include "CPool.php";
define('NODEPHP', '127.0.0.1:1337');
 
$dbh = new CPool();
$sql = "SELECT * FROM USERS";
$stmt = $dbh->prepare($sql);
 
echo $stmt->getId();
```

And another script (another http request for example) to fetch the resultset. Notice that we can execute this script all the times that we want because the compiled statement persists in the node.php server (we don't need to create it again and again within each request).

```php
include "CPool.php";
define('NODEPHP', '127.0.0.1:1337');
 
$stmt = new CPoolStatement();
$stmt->setId(1);
 
$stmt->execute();
$data = $stmt->fetchAll();
print_r($data);
```

And basically that was my sunday afternoon experiment. As you can imagine the library is totally unstable. It’s only one experiment. We can add transaccions, comits, rollbacks, savepoints, … but I needed a walk and I stopped:). What do you think?

The code is available at [github](https://github.com/gonzalo123/CPool)
