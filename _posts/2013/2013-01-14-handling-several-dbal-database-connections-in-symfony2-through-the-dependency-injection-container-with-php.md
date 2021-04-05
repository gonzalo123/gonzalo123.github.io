---
layout: post
title: "Handling several DBAL Database connections in Symfony2 through the Dependency Injection Container with PHP"
date: "2013-01-14"
tags: 
  - "databases"
  - "dbal"
  - "pdo"
  - "php"
  - "symfony2"
  - "technology"
  - "dic"
  - "pdo"
  - "symfony"
---

(This post is the second part of my previous post: Handling several PDO Database connections in Symfony2 through the Dependency Injection Container with PHP. You can read it [here](http://gonzalo123.com/2013/01/07/handling-several-pdo-database-connections-in-symfony2-through-the-dependency-injection-container-with-php/))

OK. We can handle PDOs connections inside a Symfony2 application, but what happens if we prefer [DBAL](http://www.doctrine-project.org/projects/dbal.html). As we know [DBAL](http://gonzalo123.com/2011/07/11/database-abstraction-layers-in-php-pdo-versus-dbal/ "Database abstraction layers in PHP. PDO versus DBAL") is built over PDO and adds a set of "extra" features to our database connection. It's something like PDO with steroids.

If we read the documentation, we will see how to use DBAL:

```php
<?php
$config = new \Doctrine\DBAL\Configuration();
//..
$connectionParams = array(
    'dbname' => 'mydb',
    'user' => 'user',
    'password' => 'secret',
    'host' => 'localhost',
    'driver' => 'pdo_mysql',
);
$conn = DriverManager::getConnection($connectionParams, $config);
```

As we can see to obtain a DBAL connection we use a factory method in DriverManager class. We can easily implements it in our service container:

```yaml
# databases.yml
parameters:
  doctrine.dbal.configuration: Doctrine\DBAL\Configuration
  doctrine.dbal.drivermanager: Doctrine\DBAL\DriverManager
 
  database.db1:
    driver: pdo_sqlite
    memory: true
  database.db2:
    driver: pdo_pgsql
    dbname: testdb
    user: username
    password: password
    host: 127.0.0.1
 
services:
  dbal_configuartion:
    class: %doctrine.dbal.configuration%
  db1:
    factory_class: %doctrine.dbal.drivermanager%
    factory_method: getConnection
    arguments: [%database.db1%]
  db2:
      factory_class: %doctrine.dbal.drivermanager%
      factory_method: getConnection
      arguments: [%database.db2%]
```

But if we run again our example Symfony will throws us one error:

> RuntimeException: Please add the class to service "db1" even if it is constructed by a factory since we might need to add method calls based on compile-time checks.

If we use this service container configuration outside Symfony2 application it works (remember we can use Symfony's Dependency Injection Container outside Symfony application as a component. Example [here](http://gonzalo123.com/2012/09/03/dependency-injection-containers-with-php-when-pimple-is-not-enough/ "Dependency Injection Containers with PHP. When Pimple is not enough.")). But if we want to use it with Symfony2 we need to set the "class", even here when we only need the static constructor, so we change it to:

```yaml
# databases.yml
parameters:
  doctrine.dbal.configuration: Doctrine\DBAL\Configuration
  doctrine.dbal.drivermanager: Doctrine\DBAL\DriverManager
 
  database.db1:
    driver: pdo_sqlite
    memory: true
  database.db2:
    driver: pdo_pgsql
    dbname: testdb
    user: username
    password: password
    host: 127.0.0.1
 
services:
  dbal_configuartion:
    class: %doctrine.dbal.configuration%
  db1:
    class: %doctrine.dbal.drivermanager%
    factory_class: %doctrine.dbal.drivermanager%
    factory_method: getConnection
    arguments: [%database.db1%]
  db2:
    class: %doctrine.dbal.drivermanager%
    factory_class: %doctrine.dbal.drivermanager%
    factory_method: getConnection
    arguments: [%database.db2%]
```

And that's all. We can use DBAL instead of PDO in our database connections.

**UPDATE:**

After publishing this post someone comment me Doctrine allows us to do it "out of the box" within Symfony with its DoctrineBundle:

https://github.com/doctrine/DoctrineBundle/blob/master/Resources/doc/configuration.rst#doctrine-dbal-configuration
