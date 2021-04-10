---
layout: post
title: "Using PHP classes to store configuration data"
date: "2011-01-10"
tags: 
  - "php"
  - "technology"
  - "tips"
---

In my last projects I’m using something I think is useful and it’s not a _common_ practice in our PHP projects. That’s is the usage of a plain PHP's class for the application’s configuration. Let me explain it. Normally we use ini file for configuration. PHP has a built-in [function](http://php.net/manual/es/function.parse-ini-file.php) for parse _ini_ file. It converts the ini file into a PHP array

```ini
; our conf.ini file
DEF_APP = home
DEF_BCK = main
DEF_FCT = index
AUTH_ADAPTER = Nov\Auth\Def
 
[PG1]
driver = pgsql
dsn = "pgsql:dbname=pg1;host=localhost"
username = nov
password = nov
```

And now we use our ini file:

```php
$conf = parse_ini_file("conf.ini", true);
print_r($conf);
```

Another typical configuration file is _xml_. We also can use built-in [functions](http://es.php.net/manual/en/book.simplexml.php) within PHP when we want to parse xml files. There are also projects that use plain php files, and even [yaml](http://www.yaml.org/) files (especially [Symfony](http://www.symfony-project.org/) users)

There are many standard options. Why I prefer a different one then? I like plain PHP classes because the IDE helps me with autocompletion. The usage is quite simple. Instead of using parse\_ini\_file function we only need to include our configuration class within our project.

```php
class Conf
{
 const DEF_APP = 'index';
 const DEF_BCK = 'Index\Bck\Main\Page';
 const DEF_FCT = 'init';
 const AUTH_ADAPTER = "Nov\Auth\Def";
 
 const DB_PG1 = 'PG1';
 const DB_PG1_DRIVER   = 'pgsql';
 const DB_PG1_DSN      = 'pgsql:dbname=pg1;host=localhost';
 const DB_PG1_USERNAME = 'nov';
 const DB_PG1_PASSWORD = 'nov';
}
```

Sometimes I want to use a variable stored into the application's configuration but I don’t remember exactly the name of the variable. If I've got the configuration stored into a PHP array from a ini file (or xml, yaml, ... ). I need to open the configuration file and find the name of the variable to use in my code. With a PHP class IDEs like Netbeans of VIM will pop up an autocompletion hint with the name of the variable

An example of coding autocompletion with Netbeans
[](/assets/images/netbeans.png "netbeans")

The same with VIM

[](/assets/images/vim.png "netbeans")

And that’s all. Maybe the worse part of this technique is if our application writes the configuration file (a typical installer.php). In this case we need to build the PHP’s class file and it’s slightly more complicated than writing a simple ini file.

What do you think about it? Do you use another different way?
