---
layout: post
title: "Releasing unmanaged resources (a PHP port from C#'s \"using\" statement)"
date: "2013-09-09"
tags: 
  - "php"
  - "technology"
  - "c#"
---

Sometimes we work with instances that needs to released even when exceptions happens. Something typical when we work with resources (Files, Database connections, ...) Let me show you with an example:

Imagine this class for handling file resources: 

```php
class File
{
    private $resource;
    private $logger;
 
    public function __construct($filename, Loger $logger)
    {
        $this->logger = $logger;
        $this->resource = fopen($filename);
    }
 
    public function write($string)
    {
        fwrite($this->resource, $string);
    }
 
    public function close()
    {
        $this->logger->log("file closed")   
        fclose($this->resource);
    }
}
```

We can use this class: 

```php
$file = new File(__DIR__ . "/file.txt", 'w');
$file->write("Hello\n");
// ...
// some other things
// ...
$file->write("Hello\n");
$file->close();
```

OK. What happens if one exception happens inside "some other things"? Simple answer: close() function isn't called. This can be a problem. We can face this problem with a try/catch block like this: 

```php
try {
    $file->write("Hello\n");
    // ...
    // some other things
    // ...
    $file->write("Hello\n");
    $file->close();
} catch (\Exception $e) {
    $file->close();
}
```

Or even if we're using PHP5.5 we can use "finally" keyword 

```php
try {
    $file->write("Hello\n");
    // ...
    // some other things
    // ...
    $file->write("Hello\n");
} catch (\Exception $e) {
} finally {
    $file->close();
}
```

Sometimes I need collaborate with C# projects. C# is a great language. I really like it. It has a really cool feature to solve this problem: the "[using](http://msdn.microsoft.com/en-us//library/yh598w02(v=vs.90).aspx)" statement. Because of that we are going to build today one small library to implement something similar in PHP.

First we will add G\\IDisposable interface to our File class 

```php
namespace G;
 
interface IDisposable
{
    public function dispose();
}
```

Now our File class looks like this: 

```php
use G\IDisposable;
 
class File implements IDisposable
{
    private $resource;
 
    public function __construct($filename, $mode)
    {
        $this->resource = fopen($filename, $mode);
    }
 
    public function write($string)
    {
        fwrite($this->resource, $string);
    }
 
    public function close()
    {
        fclose($this->resource);
    }
 
    public function dispose()
    {
        $this->close();
    }
}
```

And we can use our "using" function in PHP: 

```php
using(new File(__DIR__ . "/file.txt", 'w'), function (File $file) {
        $file->write("Hello\n");
        $file->write("Hello\n");
        $file->write("Hello\n");
    });
```

As we can see we can forget to close() our file instance. "using" will do it for us, even if one exception is triggered inside. We need to take into account that "using" is not a way to handle exceptions. If you want to handle them you need to do it. The only responsibility of "using" is to ensure that dispose() is been called.

We also can use an array of instances (implementing the IDisposable interface of course) 

```php
using([new Bar, new Foo], function (Bar $bar, Foo $foo) {
        echo $bar->hello("Gonzalo");
        echo $foo->hello("Gonzalo");
    });
```

And that's all. Library is available in [github](https://github.com/gonzalo123/using) and also you can use it with [composer](https://packagist.org/packages/gonzalo123/using).

**Update!** 
I've changed the Example. The first example wasn't a good one. Now File::close() closes the resource and also write a log. With the first example (without the logger) it wasn't important to close the resorce (PHP closes it).
