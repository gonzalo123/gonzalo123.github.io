---
layout: post
title: "Using a stream wrapper to access CouchDb attachments with PHP"
date: "2010-09-06"
tags: 
  - "php"
  - "couchDB"
categories: 
  - "php"
  - "couchDB"
---

I'm still working in my filesystem with CouchDb. After creating a library to enable working with PHP and CouchDB (see the post [here](http://gonzalo123.wordpress.com/2010/08/30/using-couchdb-as-filesystem-with-php/)), and after using [Monkey Patching](http://gonzalo123.wordpress.com/2010/09/01/using-monkey-patching-to-store-files-into-couchdb-using-the-standard-filesystem-functions-with-php/) to override standard PHP’s filesystem functions. I’ve created another solution now. Thanks to a [comment](http://gonzalo123.wordpress.com/2010/09/01/using-monkey-patching-to-store-files-into-couchdb-using-the-standard-filesystem-functions-with-php/#comment-377) in my last post (many thanks [Benjamin](http://www.whitewashing.de/)) I’ve discovered that it’s possible to create a stream wrapper in [PHP](http://www.php.net/manual/en/stream.streamwrapper.example-1.php) (I thought it was only available with a C extension).

It’s pretty straightforward to create the wrapper. Of course it’s only an approach. We can create more functionality to our stram wrapper but at least this example meets my needs.

```php
namespace Nov\CouchDb\Fs;
 
class StreamWrapper {
    var $position;
    var $varname;
    /**
     * @var \Nov\CouchDb\Fs\Resource
     */
    var $fs;
 
    /**
     * @param string $path
     * @return \Nov\CouchDb\Fs
     */
    private static function _getFs($path, $_url)
    {
        return \Nov\CouchDb\Fs::factory(FSCDB, $_url['host']);
    }
 
    function stream_open($path, $mode, $options, &$opened_path)
    {
        $_url = parse_url($path);
        $_path = $_url['path'];
        $fs = self::_getFs($path, $_url);
        $_type = strtolower(substr($mode, 0, 1));
        switch ($_type) {
            case 'r':
                $this->fs = $fs->open($_path);
                break;
            case 'w':
                $this->fs = $fs->open($_path, true, true);
                break;
        }
        return true;
    }
 
    function stream_read($count=null)
    {
        if (is_null($count)) {
            return $this->fs->raw();
        } else {
            return $this->fs->raw(0, $count);
        }
    }
 
    function stream_write($data, $lenght=null)
    {
        if (is_null($lenght)) {
            $this->fs->write($data);
        } else {
            $this->fs->write(mb_substr($data, 0, $lenght));
        }
        return strlen($data);
    }
 
    public function unlink($path)
    {
        $_url = parse_url($path);
        $_path = $_url['path'];
 
        $fs = self::_getFs($path, $_url)->open($_path);
        $fs->delete();
    }
 
    public function url_stat($path , $flags)
    {
        $_url = parse_url($path);
        $_path = $_url['path'];
        $fs = self::_getFs($path, $_url)->open($_path);
 
        //tamaño en bytes
        $size = $fs->getLenght();
        $out[7] = $size;
        $out['size'] = $size;
        return $out;
    }
```

That’s very simple. It’s almost the same code than the Monkey Patching example. And now if we want to use it:

```php
use Nov\CouchDb;
use Nov\CouchDb\Fs;
require_once ("Nov/Loader.php");
Nov\Loader::init();
// I use FSCDB to set up the connection parameters to our couchDb database
define('FSCDB', \NovConf::CDB1);
 
stream_wrapper_register("novCouchDb", "Nov\CouchDb\Fs\StreamWrapper") or die("Failed to register protocol");
$file = "novCouchDb://fs/home/gonzalo/new.txt";
 
$f = fopen($file, 'w+');
fwrite($f, "***12345dkkydd678901");
fclose($f);
 
$f = fopen($file, 'r');
$a = fread($f, filesize($file));
fclose($f);
 
unlink($file);
echo $a;
```

As we can see the URI is:

**novCouchDb://fs/home/gonzalo/new.txt.**

That's mean \[protocol\]://\[database\]/\[path\]

The behaviour of this example is exactly the same than Monkey Patching one but I preffer this one. It looks like more "clean" than Monkey Patching approach.

The full source code is available [here](http://code.google.com/p/nov-framework/). And the example: document\_root/tests/couchdb/test7.php
