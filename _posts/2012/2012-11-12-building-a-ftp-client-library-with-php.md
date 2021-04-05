---
layout: post
title: "Building a FTP client library with PHP"
date: "2012-11-12"
tags: 
  - "ftp"
  - "php-functions"
  - "php"
  - "phpunit"
  - "technology"
---

In my daily work I need to connect very often to FTP servers. Put files, read, list and things like that. I normally use the standard PHP functions for [Ftp](http://php.net/manual/en/book.ftp.php) it's pretty straight forward to use them. Just enable it within our installation (_\--enable-ftp_) and it's ready to use them. But last Sunday it was raining again and I start with this simple library.

Lets connect to a FTP server, switch to passive mode, put a file from one real file stored in our local filesystem and delete it.

```php
use FtpLib\Ftp,
    FtpLib\File;
 
list($host, $user, $pass) = include __DIR__ . "/credentials.php";
 
$ftp = new Ftp($host, $user, $pass);
$ftp->connect();
$ftp->setPasv();
 
$file = $ftp->putFileFromPath(__DIR__ . '/fixtures/foo');
echo $file->getName();
echo $file->getContent();
$file->delete();
```

Now the same, but without a real file. We are going to create the file on-the-fly from one string:

```php
use FtpLib\Ftp,
    FtpLib\File;
 
list($host, $user, $pass) = include __DIR__ . "/credentials.php";
 
$ftp = new Ftp($host, $user, $pass);
$ftp->connect();
$ftp->setPasv();
 
$file = $ftp->putFileFromString('bar', 'bla, bla, bla');
echo $file->getName();
echo $file->getContent();
$file->delete();
```

We also can create directories, change the working directory and delete folders in the FTP server with a fluent interface (I love fluent interfaces, indeed):

```php
$ftp->mkdir('directory')
  ->chdir('directory')
  ->putFileFromString('newFile', 'bla, bla')
  ->delete();
 
$ftp->rmdir('directory');
```

And finally we can iterate files in the FTP (I must admit that this feature was the main purpose of the library) 

```php
$ftp->getFiles(function (File $file) use ($ftp) {
    switch($file->getName()) {
        case 'file1':
            $file->delete();
            break;
        case 'file2':
            $ftp->mkdir('backup')->chdir('backup')->putFileFromString($file->getName(), $file->getContent());
            break;
    }
});
```

And that's all. You can find the library in [github](https://github.com/gonzalo123/ftplib) and you also can use it with [composer](https://packagist.org/packages/gonzalo123/ftplib).

```
"gonzalo123/ftplib": "dev-master"
```

You can also see usage examples within the [unit tests](https://github.com/gonzalo123/ftplib/blob/master/tests/FtpTest.php)
