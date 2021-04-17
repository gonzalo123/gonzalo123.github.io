---
layout: post
title: "Using Monkey Patching to store files into CouchDb using the standard filesystem functions with PHP"
date: "2010-09-01"
tags: 
  - "php"
  - "couchDB"
categories: 
  - "php"
  - "couchDB"
---

Let’s go a little further in our CouchDb storage system with PHP. I will show you why. We have some files using PHP’s [filesystem](http://www.php.net/manual/en/ref.filesystem.php) functions and we want to change them to the new interface with the [Nov\\CouchDb\\Fs library](http://code.google.com/p/nov-framework/source/browse/lib/Nov/CouchDb/Fs.php). We’ve got the following code:

```php
// Open a new file and write a string
$f = fopen($file, 'w+');
fwrite($f, "***12345dkkydd678901");
fclose($f);
 
// read a file into a variable
$f = fopen($file, 'r');
$a = fread($f, filesize($file));
fclose($f);
 
// Delete the file
unlink($file);
```

We can start to rewrite the code according to the new Nov\\CouchDb\\Fs interface. It’s not difficult but the syntax is very different so this piece of code needs almost a full rewrite. If we’ve got this kind of code split among files, we need to rewrite a lot. And rewrite always the same code is not funny so let’s do something different.

Since PHP5.3 a new design pattern is available for us: [Monkey Patching](http://en.wikipedia.org/wiki/Monkey_patch). With this pattern we can override PHP’s core functions with a home-made functions in a different namespace (another example [here](http://till.klampaeckel.de/blog/archives/105-Monkey-patching-in-PHP.html)). That’s means if I have fopen function in the above example, PHP uses the filesystem function “fopen” but if we set a namespace in our example, PHP will search first the function within the current namespace.

Using this idea I have created the following file:

```php
<?php
namespace Nov\CouchDb\Fs\Monkeypatch;
use Nov\CouchDb\Fs;
use Nov\CouchDb;
 
function unlink($filename)
{
    $fs = CouchDb\Fs::factory(FSCDB);
    $fs->delete($filename);
}
/**
 * @param string $filename
 * @param string $type
 * @return \Nov\CouchDb\Fs\Resource
 */
function fopen($filename, $type='r')
{
    $out = null;
    $fs = CouchDb\Fs::factory(FSCDB);
    $_type = strtolower(substr($type, 0, 1));
    switch ($_type) {
        case 'r':
            $out = $fs->open($filename);
            break;
        case 'w':
            $out = $fs->open($filename, true, true);
            break;
    }
    return $out;
}
function filesize($filename)
{
    $fs = CouchDb\Fs::factory(FSCDB);
    $res = $fs->open($filename);
    return $res->getLenght();
}
function fwrite(\Nov\CouchDb\Fs\Resource &$f, $data, $lenght=null)
{
    if (is_null($lenght)) {
        $out = $f->write($data);
    } else {
        $out = $f->write(mb_substr($data, 0, $lenght));
    }
    $path = $f->getPath();
    $fs = CouchDb\Fs::factory(FSCDB);
    $f = $fs->open($path);
}
function fclose(\Nov\CouchDb\Fs\Resource $f)
{
}
function fread(\Nov\CouchDb\Fs\Resource $f, $lenght=null)
{
    if (is_null($lenght)) {
        return $f->raw();
    } else {
        return $f->raw(0, $lenght);
    }
}
```

As you can see this file redefines some function into Nov\\CouchDb\\Fs\\Monkeypatch namespace. That’s means if we properly include this file into our project, when we’re under this namespace this namespace PHP will go to the new function set instead to the original one. It’s something like overriding main functions at runtime.

Maybe is more clear with an example. We’ll take back the first example and add the some extra lines at the beginning:

```php
// new code
namespace Nov\CouchDb\Fs\Monkeypatch;
include ("Nov/CouchDb/Fs/Monkeypatch.php");
require_once ("Nov/Loader.php");
\Nov\Loader::init();
define('FSCDB', \NovConf::CDB1);
// end of new code
 
$f = fopen($file, 'w+');
fwrite($f, "***12345dkkydd678901");
fclose($f);
 
// read a file into a variable
$f = fopen($file, 'r');
$a = fread($f, filesize($file));
fclose($f);
 
// Delete the file
unlink($file);
```

Another possibility is to place all include code into a separate file:

index.php:

```php
require_once ("Nov/Loader.php");
Nov\Loader::init();
 
define('FSCDB', \NovConf::CDB1);
include ("Nov/CouchDb/Fs/Monkeypatch.php");
 
include('main.php');
```

And main.php:

```php
namespace Nov\CouchDb\Fs\Monkeypatch;
 
// Open a new file and write a string
$f = fopen($file, 'w+');
fwrite($f, "***12345dkkydd678901");
fclose($f);
 
// read a file into a variable
$f = fopen($file, 'r');
$a = fread($f, filesize($file));
fclose($f);
 
// Delete the file
unlink($file);
```

And that’s all. Thanks to PHP5.3’s namespaces we can change the default behaviour of our script into a new one only setting up a namespace.

The source code can be downloaded from [here](http://code.google.com/p/nov-framework/). The examples of this post are are: document _root/tests/couchdb/test3.php, document _root/tests/couchdb/test5.php and document _root/tests/couchdb/test6.php
