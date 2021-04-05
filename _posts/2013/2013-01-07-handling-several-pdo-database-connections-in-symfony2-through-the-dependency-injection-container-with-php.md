---
layout: post
title: "Handling several PDO Database connections in Symfony2 through the Dependency Injection Container with PHP"
date: "2013-01-07"
tags: 
  - "dic"
  - "pdo"
  - "php"
  - "symfony2"
  - "technology"
  - "php-project"
  - "symfony"
---

I'm not a big fan of ORMs, especially in PHP world when all dies at the end of each request. Plain SQL is easy to understand and very powerful. Anyway in PHP we have Doctrine. Doctrine is a amazing project, probably (with permission of Symfony2) the most advanced PHP project, but I normally prefer to work with SQL instead of Doctrine.

Symfony framework is closely coupled to Doctrine and it's very easy to use the ORM from our applications. But as I said before I prefer not to use it. By the other hand I have another problem. Due to my daily work I need to connect to different databases (not only one) in my applications. In Symfony2 we normally configure the default database in our **parameters.yml** file:

```yaml
# parameters.yml
parameters:
    database_driver: pdo_pgsql
    database_host: localhost
    database_port: 5432
    database_name: symfony
    database_user: username
    database_password: password
```

 Ok. If we want to use PDO objects with different databases, we can use something like that:

```yaml
# parameters.yml
parameters:
  database.db1.dsn: sqlite::memory:
  database.db1.username: username
  database.db1.password: password
 
  database.db2.dsn: pgsql:host=127.0.0.1;port=5432;dbname=testdb
  database.db2.username: username
  database.db2.password: password
```

And now create the PDO objects within our code with new \\PDO():

```php
$dsn      = $this->container->getParameter('database.db1.dsn');
$username = $this->container->getParameter('database.db1.username');
$password = $this->container->getParameter('database.db1.password')
 
$pdo = new \PDO($dsn, $username, $password);
```

It works, but it's awful. We store the database credentials in the service container but we aren't using the service container properly. So we can do one small improvement. We will create a new configuration file called **databases.yml** and we will include this new file within the **services.yml**:

```yaml
# services.yml
imports:
- { resource: databases.yml }
```

And create our databases.yml: \

```yaml
# databases.yml
parameters:
  db.class: Gonzalo123\AppBundle\Db\Db
 
  database.db1.dsn: sqlite::memory:
  database.db1.username: username
  database.db1.password: password
 
  database.db2.dsn: pgsql:host=127.0.0.1;port=5432;dbname=testdb
  database.db2.username: username
  database.db2.password: password
 
services:
  db1:
    class: %db.class%
    calls:
      - [setDsn, [%database.db1.dsn%]]
      - [setUsername, [%database.db1.username%]]
      - [setPassword, [%database.db1.password%]]
  db2:
    class: %db.class%
    calls:
      - [setDsn, [%database.db2.dsn%]]
      - [setUsername, [%database.db2.username%]]
      - [setPassword, [%database.db2.password%]]
```

As we can see we have created two new services in the dependency injection container called db1 (sqlite in memory) and db2 (one postgreSql database) that use the same class (in this case 'Gonzalo123\\AppBundle\\Db\\Db'). So we need to create our Db class:

```php
<?php
 
namespace Gonzalo123\AppBundle\Db;
 
class Db
{
    private $dsn;
    private $username;
    private $password;
 
    public function setDsn($dsn)
    {
        $this->dsn = $dsn;
    }
 
    public function setPassword($password)
    {
        $this->password = $password;
    }
 
    public function setUsername($username)
    {
        $this->username = $username;
    }
 
    /** @return \PDO */
    public function getPDO()
    {
        $options = array(\PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION);
        return new \PDO($this->dsn, $this->username, $this->password, $options);
    }
}
```

And that's all. Now we can get a new PDO object from our service container with:

```php
$this->container->get('db1')->getPDO();
```

Better, isn't it? But it's still ugly. We need one extra class (Gonzalo123\\AppBundle\\Db\\Db) and this class creates a new instance of PDO object (with getPDO()). Do we really need this class? the answer is no. We can change our service container to:

```yaml
# databases.yml
parameters:
  pdo.class: PDO
  pdo.attr_errmode: 3
  pdo.erromode_exception: 2
  pdo.options:
    %pdo.attr_errmode%: %pdo.erromode_exception%
 
  database.db1.dsn: sqlite::memory:
  database.db1.username: username
  database.db1.password: password
 
  database.db2.dsn: pgsql:host=127.0.0.1;port=5432;dbname=testdb
  database.db2.username: username
  database.db2.password: password
 
services:
  db1:
    class: %pdo.class%
    arguments:
      - %database.db1.dsn%
      - %database.db1.username%
      - %database.db1.password%
      - %pdo.options%
  db2:
    class: %pdo.class%
    arguments:
      - %database.db2.dsn%
      - %database.db2.username%
      - %database.db2.password%
      - %pdo.options%
```

Now we don't need getPDO() and we can get the PDO object directly from service container with: 

```php
$this->container->get('db1');
```

And we can use something like this within our controllers (or maybe better in models):

```php
<?php
 
namespace Gonzalo123\AppBundle\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
 
class DefaultController extends Controller
{
    public function indexAction($name)
    {
        // this code should be out from controller, in a model object.
        // It is only an example
        $pdo = $this->container->get('db1');
        $pdo->exec("CREATE TABLE IF NOT EXISTS messages (id INTEGER PRIMARY KEY, title TEXT, message TEXT)");
        $pdo->exec("INSERT INTO messages(id, title, message) VALUES (1, 'title', 'message')");
        $data = $pdo->query("SELECT * FROM messages")->fetchAll();
        //
 
        return $this->render('AppBundle:Default:index.html.twig', array('usuario' => $data));
    }
}
```
