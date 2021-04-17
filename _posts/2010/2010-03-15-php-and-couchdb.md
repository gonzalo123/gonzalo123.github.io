---
layout: post
title: "PHP and couchDB"
date: "2010-03-15"
tags: 
  - "php"
  - "couchDB"
categories: 
  - "php"
  - "couchDB"
---

Last days I've been playing with [couchDB](http://couchdb.apache.org/) and PHP. There are several ways to use [couchDB](http://wiki.apache.org/couchdb/Getting_started_with_PHP) [with PHP](http://wiki.apache.org/couchdb/Getting_started_with_PHP). There also are extensions but I want to build a simple library and I will show now the code.

CouchDB has a great RestFul API so I we want to use CouchDB we only need to perform http requests. I have done a small class to do the requests. You can see the code [here](http://code.google.com/p/gam-http/). It's a simple class that uses PHP's curl functions.

I come from relational database world. [NoSQL](http://blog.oskarsson.nu/2009/06/nosql-debrief.html) is new for me. Maybe I'm wrong but I want to use INSERT, UPDATE, DELETE and SELECT statements in CouchDB in the same way I use them in Relational database.

The class is focused in the HTTP Document API. There is a great tutorial [here](http://wiki.apache.org/couchdb/HTTP_Document_API) that explains the API. Now I'll show the interface I've made to perform the statements with CouchDB.

The examples assumes there are CouchDB host at localhost:5987. Also as I've said before the class focused only in HTTP Document API so first of all I create a new database called users using CouchDB's web interface (http://localhost:5984/ \utils/). Now let's start:

### INSERT

```php
$couchDb = new Nov_CouchDb('localhost', 5984);
$couchDb->db('users')->insert('gonzalo', array('password' => "g1"));
```

### SELECT

```php
$data = $couchDb->db('users')->select('gonzalo')->asArray();
print_r($data);
```

### UPDATE

```php
$couchDb->db('users')->update('gonzalo', array('password' => 'g2'));
```

### DELETE

```php
$out = $couchDb->db('users')->delete('gonzalo')->asArray();
print_r($out);
```

Basically Nov_CouchDb has a constructor that stores connection information into private member variables:

```php
class Nov_CouchDb
{
    private $_protocol;
    private $_host;
    private $_port;
    private $_user;
    private $_password;
 
    public function __construct($host, $port=Nov_Http::DEF_PORT , $protocol=Nov_Http::HTTP, $user = null, $password=null)
    {
        $this->_host     = $host;
        $this->_port     = $port;
        $this->_protocol = $protocol;
        $this->_user     = $user;
        $this->_password = $password;
    }
}
```

There are also a public function called db. This function stores the database and returns the instance of the class (I like chainable functions)

```php
...
    private $_db;
    /**
     * @param string $db
     * @return Nov_CouchDb
     */
    public function db($db)
    {
        $this->_db = $db;
        return $this;
    }
...
```

And finally the main functions:

### SELECT

```php
public function select($key)
{
    try {
        $out = Nov_Http::connect($this->_host, $this->_port, $this->_protocol)
            ->setCredentials($this->_user, $this->_password)
            ->doGet("{$this->_db}/{$key}");
    } catch (Nov_Http_Exception $e) {
        $this->_manageExceptions($e);
    }
    return new Nov_CouchDb_Resulset($out);
}
```

### INSERT

```php
/**
 * @param string $key
 * @param array $values
 * @return Nov_CouchDb_Resulset
 */
public function insert($key, $values)
{
    try {
        $out = Nov_Http::connect($this->_host, $this->_port, $this->_protocol)
            ->setCredentials($this->_user, $this->_password)
            ->setHeaders(array('Content-Type' =>  'application/json'))
            ->doPut("{$this->_db}/{$key}", json_encode($values));
    } catch (Nov_Http_Exception $e) {
        $this->_manageExceptions($e);
    }
    return new Nov_CouchDb_Resulset($out);
}
```

### UPDATE

```php
/**
 * @param string $key
 * @param array $values
 * @return Nov_CouchDb_Resulset
 */
public function update($key, $values)
{
    try {
        $http = Nov_Http::connect($this->_host, $this->_port, $this->_protocol)
            ->setCredentials($this->_user, $this->_password);
        $out = $http->doGet("{$this->_db}/{$key}");
        $reg = json_decode($out);
        $out = $http->setHeaders(array('Content-Type' =>  'application/json'))
            ->doPut("{$this->_db}/{$key}", json_encode($reg));
    } catch (Nov_Http_Exception $e) {
        $this->_manageExceptions($e);
    }
    return new Nov_CouchDb_Resulset($out);
}
```

### DELETE

```php
/**
 * @param string $key
 * @return Nov_CouchDb_Resulset
 */
public function delete($key)
{
    try {
        $http = Nov_Http::connect($this->_host, $this->_port, $this->_protocol)
            ->setCredentials($this->_user, $this->_password);
        $out = $http->doGet("{$this->_db}/{$key}");
        $reg = json_decode($out);
        $out = $http->doDelete("{$this->_db}/{$key}", array('rev' => $reg->_rev));
    } catch (Nov_Http_Exception $e) {
        $this->_manageExceptions($e);
    }
    return new Nov_CouchDb_Resulset($out);
}
```

insert, update, delete and select returns an instance of Nov \CouchDb \Resulset. CouchDb API returns a json string. But sometimes I want a PHP array or maybe an object. I use Nov \CouchDb \Resulset to perform those transformations:

```php
// Different outputs
$data = $couchDb->db('users')->select('dummy')->asArray();
print_r($data);
 
$data = $couchDb->db('users')->select('dummy')->asObject();
print_r($data);
 
$data = $couchDb->db('users')->select('dummy')->asJson();
print_r($data);
```

### EXCEPTIONS

CouuchDB API is a RestFul API so it uses HTTP exceptions when something wrong happens. I've done three exceptions. One generic and other two with the most common exception in relational databases: NoDataFound and DupValOnIndex. this is because I want do things like the following ones:

```php
try {
    $couchDb->db('users')->update('gonzalo', array('password' => 'g2'));
} catch (Nov_CouchDb_Exception_NoDataFound $e) {
    echo "No data found \n";
}
 
$couchDb->db('users')->insert('gonzalo1', array('password' => "g1"));
try {
    $couchDb->db('users')->insert('gonzalo1', array('password' => "g1"));
} catch (Nov_CouchDb_Exception_DupValOnIndex $e) {
    echo "Dup Val On Index \n";
}
try {
    $couchDb->db('users')->delete('gonzalo');
} catch (Nov_CouchDb_Exception_NoDataFound $e) {
    echo "No data found \n";
}
```

The source code is available on google code [here](http://code.google.com/p/nov-couchdb/)
