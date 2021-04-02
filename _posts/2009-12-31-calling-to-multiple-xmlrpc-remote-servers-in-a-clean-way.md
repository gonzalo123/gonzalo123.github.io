---
layout: post
permalink: /:year/:month/:day/:title/
title: "Clean way to call to multiple xmlrpc remote servers"
date: "2009-12-31"
categories: 
  - "php"
  - "technology"
  - "web-development"
---

## The problem:

I've got a class library in PHP. This class library is distributed in a several servers and I want to call them synchronously. The class library has a XMLRPC interface with [Zend Framework](http://www.framework.zend.com/manual/en/zend.xmlrpc.server.html) [XMLRPC](http://www.framework.zend.com/manual/en/zend.xmlrpc.server.html) [server](http://www.framework.zend.com/manual/en/zend.xmlrpc.server.html). There is an example class

```php
class Gam_Dummy
{
    /**
     * foo
     *
     * @param integer $arg1
     * @param integer $arg2
     * @return integer
     */
    function foo($arg1, $arg2)
    {
        return $arg1 + $arg2;
    }
}
```

and the xmlrpc server:

```php
$class = (string) $_GET['class'];
$server = new Zend_XmlRpc_Server();
$server->setClass($class);
echo $server->handle();
```

## First solution

An easy a fast solution for calling remote interfaces is:

```php
$class = "Gam_Dummy";
$client = new Zend_XmlRpc_Client("http://location/of/xmlrpc/server?class={class}");
echo $client->call('foo', array($arg1, $arg2));

```

and if we have several remote servers:

```php
$servers = array(
    'server1' => 'http://location/of/xmlrpc/server1',
    'server2' => 'http://location/of/xmlrpc/server2',
    'server3' => 'http://location/of/xmlrpc/server3'
    );
 
$class = "Gam_Dummy";
foreach (array_values($servers) as $_server) {
    $server = "{_server}?class={$class}";
    $client = new Zend_XmlRpc_Client($server);
    echo $client->call('foo', array($arg1, $arg2));
}
```

## Second solution (one remote server):

I want to use the following interface to call my remote class

```php
$class = "Gam_Dummy";
Gam_Dummy::remote("Gam_Dummy", 'server1')->foo($arg1, $arg2);
```

Why? The answer is because I like to coding with the help of the IDE. If I use the first solution I must remember Gam\_Dummy class has a foo function with two parameters. With the second solution if I place the [PHPDoc](http://www.phpdoc.org/) code correctly my IDE will help me showing me the function list of the Gam\_Dummy class and even when I type Gam\_ IDE will show me all the classes of my repository starting with Gam\_. That issue could sound irrelevant for a lot of people but for me is really useful

To get this interface I will change my Gam\_Dummy class to:

```php
class Gam_Dummy
{
    /**
     * foo
     *
     * @param integer $arg1
     * @param integer $arg2
     * @return integer
     */
    function foo($arg1, $arg2)
    {
        return $arg1 + $arg2;
    }
 
    /**
     * Remote interface
     *
     * @param string|array $server
     * @return Gam_Dummy
     */
    static function remote($server)
    {
        return new Remote(get_called_class(), $server);
    }
}
```

And of course Remote class:

```php
class Remote
{
    private $_class  = null;
    private $_server = null;
 
    function __construct($class, $server)
    {
        $this->_class  = $class;
        $this->_server = $server;
    }
 
    function __call($method, $arguments)
    {
        if (class_exists($this->_class)) {
            $server = "{$this->_server}?class={$this->_class}";
            $client = new Zend_XmlRpc_Client($server);
            return $client->call($method, array($arg1, $arg2));
        }
    }
}
```

Cool. Isn't it?. But there is a problem if I want to work with two or more remote servers I must write one line of code for each server:

```php
Gam_Dummy::remote('http://location/of/xmlrpc/server1')->foo($arg1, $arg2);
Gam_Dummy::remote('http://location/of/xmlrpc/server2')->foo($arg1, $arg2);
Gam_Dummy::remote('http://location/of/xmlrpc/server3')->foo($arg1, $arg2);

```

or may better with the array $servers

```php
foreach (array_values($servers) as $server) {
    Gam_Dummy::remote($server)->foo($arg1, $arg2);
}
```

## Third solution for multiple remote servers:

I would like to use this interface instead of solution two with a _foreach_ for multiple servers:

```php
$servers = array(
    'server1' => 'http://location/of/xmlrpc/server1',
    'server2' => 'http://location/of/xmlrpc/server2',
    'server3' => 'http://location/of/xmlrpc/server3'
    );
 
Gam_Dummy::remote($servers)->foo($arg1, $arg2);
```

so I change Remote class to:

```php
class Remote
{
    private $_class  = null;
    private $_server = null;
 
    function __construct($class, $server)
    {
        $this->_class  = $class;
        $this->_server = $server;
    }
 
    function __call($method, $arguments)
    {
        $out = array();
        if (is_array($this->_server)) {
            foreach ($this->_server as $key => $_server) {
                $server = "{$_server}?class={$this->_class}";
                $client = new Zend_XmlRpc_Client($server);
                $out[$key] = $client->call($method, $arguments);
            }
        } else {
            $server = "{$this->_server}?class={$this->_class}";
            $client = new Zend_XmlRpc_Client($server);
            $out = $client->call($method, $arguments);
        }
        return $out;
    }
}
```
