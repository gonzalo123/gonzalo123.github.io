---
layout: post
title: "How to rewrite urls with PHP 5.4's built-in web server"
date: "2012-10-15"
tags: 
  - "php"
  - "php5-4"
  - "technology"
---

PHP 5.4 comes with a flaming built-in web server. This server is (obviously) not suitable to use in production environments, but it's great if we want to check one project quickly:

- git clone from github
- composer install to install dependencies
- run the built-in web server and test the application.

```commandline
php -S localhost:8888 -t www/
```

But is very usual to use mod\_rewrite or similar to send all requests to the front controller. With apache is pretty straight forward to do it:

```apache
<IfModule mod_rewrite.c>
    Options -MultiViews
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ index.php [QSA,L]
</IfModule>
```

But, does it work with the built-in web server? The answer is yes, but with another syntax. We only need to create one router file and start our server with this router:

```php
<?php
// www/routing.php
if (preg_match('/\.(?:png|jpg|jpeg|gif)$/', $_SERVER["REQUEST_URI"])) {
    return false;
} else {
    include __DIR__ . '/index.php';
}
```

And now we start the server with:

```commandline
php -S localhost:8888 www/routing.php
```

Easy, isn't it?
