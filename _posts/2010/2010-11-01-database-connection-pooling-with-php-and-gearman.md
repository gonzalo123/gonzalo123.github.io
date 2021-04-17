---
layout: post
title: "Database connection pooling with PHP and gearman"
date: "2010-11-01"
tags: 
  - "php"
  - "postgreSQL"
  - "gearman"
categories: 
  - "php"
  - "postgreSQL"
  - "gearman"
---

Handling Database connections with PHP is pretty straightforward. Just open database connection, perform the actions we need, and close it. There’s a little problem. We cannot create a pull of database connections. We need to create the connection in every request. Create and destroy, again and again. There are some third-party solutions like [SQL relay](http://sqlrelay.sourceforge.net/) or [pgpool2](http://pgpool.projects.postgresql.org/) (if you use PostgreSQL like me). In this post I’m going to try to explain a personal experiment to create a connection pooling using [gearman](http://gearman.org/). Another purpose of this experiment is create a prepared statements' cache not only for the current request but also for all ones. Let’s start.

This is an example of **SELECT** statement with PDO and PHP

```php
$dbh = new PDO('pgsql:dbname=pg1;host=localhost', 'user', 'pass');
$stmt = $dbh->prepare($sql);
$stmt->execute();
$data = $stmt->fetchAll();
```

Basically the idea is to use a gearman worker to perform every database operations. As far as I known we cannot pass PDO instances from gearman worker to gearman client. Even with object serialization (PDO objects cannot be serialized). That’s a problem.

My idea is use the same interface than using PDO but let the database work to the worker and obtaining a connection id instead of a real PDO connection.

That’s is the **configuration** class. We can see It’s defined one database connection and two gearman servers at the same host:

```php
class PoolConf
{
    const PG1 = 'PG1';
    static $DB = array(
        self::PG1 => array(
            'dsn'      => "pgsql:dbname=gonzalo;host=localhost",
            'username' => 'user',
            'password' => 'pass',
            'options'  => null),
    );
 
    static $SERVERS = array(
        array('127.0.0.1', 4730),
        array('127.0.0.1', 4731),
    );
}
```

**We start the workers:**

```commandline
gearmand -d --log-file=/var/log/gearman --user=gonzalo -p=4730
gearmand -d --log-file=/var/log/gearman --user=gonzalo -p=4731
```

How many workers we need to start? Depends on your needs. We must realize gearman is not a pool. It’s a queue. But we can start as many servers as we want  (OK it's depends on our RAM) and create a poll of queues. We need to remember that if we start only one gearman server we only can handle only one database operation each time (it's a queue) and it will be a huge bottleneck if the application scales. So you need to assess your site and evaluate how many concurrent operations you normally have and start as many gearman server as you need.

Maybe is difficult to explain but the final outcome will be something like that:

```php
use Pool\Client;
$conn = Client::singleton()->getConnection(PoolConf::PG1);
 
$sql = "SELECT * FROM TEST.TBL1";
$stmt = $conn->prepare($sql);
 
$stmt->execute();
$data = $stmt->fetchall();
echo
```

We must take into account that $stmt is not a "real" PHP statement. The real PHP statement is stored into a static variable within the worker. Our $stmt is an instance of Pool\\Server\\Stmt class. This class has some public methods with the same name than the real statement (because of that it behaves as a real statement), and internally those methods are calls to gearman worker. The same occurs with $conn variable. It's not a real PDO connection. It's am instance of Pool\\Server\\Connection Class.

```php
// Pool/Server/Stmt.php
namespace Pool\Server;
 
class Stmt
{
    private $_smtpId = null;
    private $_cid    = null;
    private $_client = null;
 
    function __construct($stmtId, $cid, $client)
    {
        $this->_stmt = $stmtId;
        $this->_cid = $cid;
        $this->_client = $client;
    }
 
    public function execute($parameters=array())
    {
        $out = $this->_client->do('execute', serialize(array(
            'parameters' => $parameters,
            'stmt'       => $this->_stmt,
            )));
        $error = unserialize($out);
        if (is_a($error, '\Pool\Exception')) {
            throw $error;
        }
        $this->_stmt = $out;
        return $this;
    }
 
    public function fetchAll()
    {
        $data = $this->_client->do('fetchAll', serialize(array(
            'stmt' => $this->_stmt,
            )));
        return unserialize($data);
    }
}
```

The **worker**. The following code is an extract. You can see the full code [here](http://code.google.com/p/php-pool/source/browse/worker/worker.php)

```php
# Create our worker object.
$worker= new GearmanWorker();
foreach (PoolConf::$SERVERS as $server) {
    $worker->addServer($server[0], $server[1]);
}
 
\Pool\Server::init();
 
$worker->addFunction('getConnection', 'getConnection');
$worker->addFunction('prepare', 'prepare');
$worker->addFunction('execute', 'execute');
$worker->addFunction('fetchAll', 'fetchAll');
$worker->addFunction('info', 'info');
$worker->addFunction('release', 'release');
$worker->addFunction('beginTransaction', 'beginTransaction');
$worker->addFunction('commit', 'commit');
$worker->addFunction('rollback', 'rollback');
 
while (1) {
    try {
        $ret = $worker->work();
        if ($worker->returnCode() != GEARMAN_SUCCESS) {
            break;
        }
    } catch (Exception $e) {
        echo $e->getMessage();
    }
}
 
function fetchAll($job)
{
    echo __function__."\n";
    $params = unserialize($job->workload());
    $stmtId = $params['stmt'];
    return serialize(\Pool\Server::fetchAll($stmtId));
}
 
function execute($job)
{
    echo __function__."\n";
    $params = unserialize($job->workload());
    $stmtId = $params['stmt'];
    $parameters = $params['parameters'];
    return \Pool\Server::execute($stmtId, $parameters);
}
...
```

The heart of the worker is [\\Pool\\Server](http://code.google.com/p/php-pool/source/browse/lib/Pool/Server.php) class. This class performs every real PDO operations and stores statements and connections into static private variables.

And now we can use the database pool reusing connections and prepared statements. You can see [here](http://gonzalo123.wordpress.com/2010/10/05/performance-analysis-using-bind-parameters-with-pdo-and-php/) a small performance test of reusing prepared statements in an older post.

I’ve also implemented a small error handling. Errors in the worker are serialized and thrown on the client simulating the normal operation of standard PDO usage.

Now a set of examples:

**Simple queries. And a simple error handling:**

```php
include('../conf/PoolConf.php');
include('../lib/Pool/Client.php');
include('../lib/Pool/Server.php');
include('../lib/Pool/Exception.php');
include('../lib/Pool/Server/Connection.php');
include('../lib/Pool/Server/Stmt.php');
 
use Pool\Client;
$conn = Client::singleton()->getConnection(PoolConf::PG1);
 
$sql = "SELECT * FROM TEST.TBL1";
$stmt = $conn->prepare($sql);
 
$stmt->execute();
$data = $stmt->fetchall();
echo "<p>count: " . count($data) . "</p>";
 
try {
    $sql = "SELECT * TEST.NON_EXISTENT_TABLE";
    $stmt = $conn->prepare($sql);
 
    $stmt->execute();
    $data = $stmt->fetchall();
    echo "<p>count: " . count($data) . "</p>";
 
} catch (Exception $e) {
    echo "ERROR: " . $e->getMessage();
}
 
print_r(Client::singleton()->info(PoolConf::PG1));
```

**Now with bind parameters:**

```php
<?php
include('../conf/PoolConf.php');
include('../lib/Pool/Client.php');
include('../lib/Pool/Server.php');
include('../lib/Pool/Exception.php');
include('../lib/Pool/Server/Connection.php');
include('../lib/Pool/Server/Stmt.php');
 
use Pool\Client;
$conn = Client::singleton()->getConnection(PoolConf::PG1);
 
$data = $conn->prepare("SELECT * FROM TEST.TBL1 WHERE SELECCION=:S")->execute(array('S' => 1))->fetchall();
 
echo count($data);
 
print_r(Client::singleton()->info(PoolConf::PG1));
```

**And now a transaction:**

```php
<?php
include('../conf/PoolConf.php');
include('../lib/Pool/Client.php');
include('../lib/Pool/Server.php');
include('../lib/Pool/Exception.php');
include('../lib/Pool/Server/Connection.php');
include('../lib/Pool/Server/Stmt.php');
 
use Pool\Client;
$conn = Client::singleton()->getConnection(PoolConf::PG1);
 
$conn->beginTransaction();
$data = $conn->prepare("SELECT * FROM TEST.TBL1 WHERE SELECCION=:S")->execute(array('S' => 1))->fetchall();
$conn->rollback();
 
print_r(Client::singleton()->info(PoolConf::PG1));
```

**Conclusion.**

That’s a personal experiment. It works, indeed, but probably it’s crowed by bugs. You can use it in production if you are a brave developer :).

**Source code.** The full sourcecode is available [here](http://code.google.com/p/php-pool/) at google code
