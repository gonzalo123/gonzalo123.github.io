---
layout: post
title: "Flushing files with PHP"
date: "2010-07-30"
tags: 
  - "php"
categories: 
  - "php"
---

Sometimes we need to show files as pdf from PHP. That’s pretty straightforward.

```php
$buffer = null;
$f = fopen($filePath, "rb");
if ($f ) {
    $buffer.= fread($f, filesize($filePath));
}
fclose ($f);
```

And now we flush the buffer with the right headers.

```php
$filename = basename($filePath);
$type = ‘attachement’; // can be ‘innline’
 
// get mime type. finfo must be installed. PHP >= 5.3.0, PECL fileinfo >= 0.1.0
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mimeType = finfo_file($finfo, $filePath);
 
header("Expires: Wed, 20 Sep 1977 16:10:00 GMT");
header("Cache-Control: no-cache");
header('Cache-Control: maxage=3600');
header('Pragma: public');
//header("Content-Length: " .filesize($path) );
header("Content-Disposition: {$type}; filename={$filename}");
header("Content-Transfer-Encoding: binary");
header("Content-type: ".mimeType”);
 
echo $buffer;
```

Apparently the order of the headers is irrelevant but if you need to work with IE (poor guy), use those headers exactly in this order. I don’t know exactly the reason but this combination always works for me and if I overlook it and I change the order it’s likely to have problems with IE.

Another trick is the commented line.

```php
//header("Content-Length: " .filesize($path) );
```

According to standards it must be set the length of the file with the Content-Length header, but I noticed that if I don’t set this header, the browser opens associated application (Acrobat reader e.g with pdf files) earlier than if I set it. This behaviour is visible with big files. With the Content-Length header the browser opens associated application to the file when the file if fully downloaded and without this header the browser don’t wait to finish the download of the file to open it. That's means better user experience.
