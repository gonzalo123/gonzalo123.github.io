---
layout: post
title: "Protect files within public folders with mod_rewrite and PHP"
date: "2010-11-29"
tags: 
  - "php"
categories: 
  - "php"
---

Here's the problem. We have a legacy application (or a WordPress blog for the example) and we want to protect the access to the application according to our corporate single sign on. We can create a plug-in in WordPress to ensure only our single sign-on's session cookie is activated.

With WordPress we can create a simple plug-in to perform our desired behaviour:

```php
<?php
// wp-content/plugins/gamAuth.php
/**
 *  @package gamAuth
 *  @version 1.0.0
 */
/*
Plugin Name: gamAuth
Plugin URI: #
Description: Forces users to be authenticated
Author: Gonzalo Ayuso
Version: 1.0.0
Author URI: gonzalo122.wordpress.com
*/
 
function redirectExit() {;
 nocache_headers();
 header("HTTP/1.1 302 Moved Temporarily");
 header("Location: http://www.google.com");
 header("Status: 302 Moved Temporarily");
 exit();
}
function chechAuth() {
 // here we check our session
}
 
function ac_auth_redirect() {
 if (!chechAuth()) redirectExit();
}
if('wp-login.php' != $pagenow && 'wp-register.php' != $pagenow) {
 add_action('template_redirect', 'ac_auth_redirect');
}
```

This plug-in works, but what happen with the uploaded files? If we upload a PNG file and we show it in our post, our WordPress will create a direct link to the uploaded file in _wp-content/uploads_ folder. That’s means if a non-authenticated user knows the url of the link, he will see the picture, because our simple plug-in does not affect to direct links. OK WordPress is open source. We can change the code and change the behaviour of link generation. Maybe there’s a plugin to do it (don't hesitate on explain it to me, anyway) but if we don’t want to touch the WordPress code or maybe we cannot do it (because it's a legacy application), here we are a simple trick to force authorization even over files within public folders, without change any line of the original application.

The trick is very simple. First we activate mod_rewrite in our server and create a _.htaccess_ files in our _wp-content/uploads_ folder: An example with apache:

```
RewriteEngine on
RewriteBase /
RewriteRule !\.(php)$ /wordpress/wp-content/uploads/media.php
```

That’s means we are going to redirect every file request within or _upload_ folder to a PHP script called _media.php_

And now our media.php will check the authorization.

```php
<?php
$uri = $_SERVER['REQUEST_URI'];
$documentRoot = $_SERVER['DOCUMENT_ROOT'];
$filename = $documentRoot . $uri;
$pathParts = pathinfo($filename);
$mime = array(
 'jpe'  => 'image/jpeg',
 'jpeg' => 'image/jpeg',
 'jpg'  => 'image/jpeg',
 'png'  => 'image/png',
 'xls'  => 'application/vnd.ms-excel',
 'pdf'  => 'application/pdf',
 );
function chechAuth() {
 // here we check our session
}
$ext = strtolower($pathParts['extension']);
// there are built-in ways to check file mime type but
// they don't work fine, at least for me and I preffer to
// check it manually with the extension
if (is_file($filename) && array_key_exists($ext, $mime)) {
 if (chechAuth()) {
  header("Content-Type: " . $mime[$ext]);
  readfile($filename);
  exit();
 }
}
echo "HTTP/1.1 503 Service Unavailable";
header('HTTP/1.1 503 Service Unavailable', true, 503);
```

If you want to show big files such as video files, maybe you need to have a look to one of my older [posts](http://gonzalo123.com/2010/09/20/live-video-streaming-with-php/ "Live video streaming with PHP").

Now when our user tries to get wp-content/uploads/2010/11/img1.jpg from our server, the mod rewrite will forward the job internally to _media.php_ and we will check the authorizations before showing the file,  and the user will see in the browser 's URL wp-content/uploads/2010/11/img1.jpg instead of media.php.

And that’s all. We can use this technique not only to protect our files. We also can dynamically add a watermark to our images without changing the original URL, show different image quality depends on the user or block a IP range to our media files.

Of course If we start an application from scratch there are better solutions but the good point of this trick is that it’s fully compliant with our legacy application and we don’t need to change anything inside.
