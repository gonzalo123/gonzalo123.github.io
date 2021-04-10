---
layout: post
title: "Watermarks in our images with PHP and mod_rewrite"
date: "2011-02-28"
tags: 
  - "php"
  - "technology"
  - "gearman"
  - "mod_rewrite"
---

Imagine you have a set of images and you need to add watermark to all of them. You can do the work with an image editor (Gimp or Photoshop for example). It’s easy work but if your image library is big, the easy work become into hard. But imagine again you need to add a dynamic watermark, let’s say the current timestamp. To solve this problem we can use PHP’s GD image function set.

The idea is simple. Instead of opening directly the image, we are going to open a PHP script. This PHP script will open the original image file with [imagecreatefromjpeg](http://www.php.net/manual/en/function.imagecreatefromjpeg.php), add the footer and flush the new image to the browser with the properly headers.

But I don't want to rewrite all the anchor’s hrefs within my HTML files. Because of that I’m going to use the same trick I used in my one of my [posts](http://gonzalo123.wordpress.com/2010/11/29/protect-files-within-public-folders-with-mod_rewrite-and-php/). With a simple .htaccess in our jpg's folder, our apache’s mod\_rewrite will do work for us:

```
RewriteEngine on
RewriteRule !\.(php)$ watermark.php
```

And now our PHP script called watermark.php

```php
<?php
$uri = $_SERVER['REQUEST_URI'];
$documentRoot = $_SERVER['DOCUMENT_ROOT'];
$filename = $documentRoot . $uri;
if (realpath(__FILE__) == realpath($filename)) {
    exit();
}
$stringSize = 3;
$footerSize = ($stringSize==1) ? 12 : 15;
$footer = date('d/m/Y H:i:s');
 
list($width, $height, $image_type) = getimagesize($filename);
$im = imagecreatefromjpeg($filename);
imagefilledrectangle (
        $im,
        0,
        $height,
        $width,
        $height - $footerSize, imagecolorallocate($im, 49, 49, 156));
 
imagestring($im,
        $stringSize,
        $width-(imagefontwidth($stringSize)*strlen($footer)) - 2,
        $height-$footerSize,
        $footer,
        imagecolorallocate($im, 255, 255, 255));
 
header( 'Content-Type: image/jpeg' );
imagejpeg($im);
```

Simple, isn’t it? You can change my blue footer with a your logo. Easy with PHP’s gd functions. Of course you need to have your PHP compiled with [GD](http://www.php.net/manual/en/image.installation.php) support.

Original image:
[](/assets/images/original.jpg "original")

After transformation:
[](/assets/images/after.png "after")
