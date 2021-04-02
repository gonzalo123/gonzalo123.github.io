---
layout: post
permalink: /:year/:month/:day/:title/
title: "An idea for calling PostgreSQL's stored procedures with PDO"
date: "2009-12-08"
tags: 
  - "php"
  - "web-development"
---

As far as I know PDO doesn't allow to call directly the  PostgreSQL's strored procedures. That's not a problem. We can create a SQL and call a stored procedures as simple sql.

Imagine we have a stored procedure in the schema called 'schemaName' with the name 'method1'.

```postgresql
CREATE OR REPLACE FUNCTION schemaName.method1(param1 numeric, param2 numeric)
  RETURNS numeric AS
$BODY$
BEGIN
   RETURN param1 + param2;
END;
$BODY$
  LANGUAGE 'plpgsql' VOLATILE
  COST 100;
```

The way of call it is something like this:

```php
$conn = new PDO($dsn, $user, $password);
$conn->beginTransaction();
$stmt = $this->prepare("SELECT * FROM schemaName.method1(?, ?)");
$stmt->execute(1, 2);
$stmt->setFetchMode(PDO::FETCH_ASSOC);
$out = $stmt->fetchAll();
$conn->commit();
```

An idea for doing the same in a more clean way is:

```php
$conn = new MyPDO($dsn, $user, $password);
$conn->beginTransaction();
$out = $conn->setSchema('schemaName')->method1(1, 2);
$conn->commit();
```

That's only an approach. I haven't think a lot about it but that's OK as a first approach.

And now the class I've created extending PDO to obtain the above interface.

The trick is in __call function. Using __call I have dynamic functions in my MyPDO class and I will suppose every functions will be stored procedures.

```php
class MyPDO extends PDO
{
    private $_schema = null; 
 
    /**
     * Set Schema
     *
     * @return MyPDO
     */
    public function setSchema($_squemaName)
    {
        $this->_schema = $_squemaName;
        return $this;
    } 
 
    function __call($method, $arguments)
    {
        $_params = array();
        if (count($arguments)>0) {
            for ($i=0; $i<count($arguments); $i++) {
                $_params[] = '?';
            }
        } 
 
        $stmt = $this->prepare("SELECT * FROM {$this->_schema}.{$method}(" .
            implode(', ', $_params) .  ")");
        $stmt->execute($arguments);
        $stmt->setFetchMode(PDO::FETCH_ASSOC);
        return $stmt->fetchAll();
    }
}
```
