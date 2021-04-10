---
layout: post
title: "Speed up page page load combining javascript files with PHP"
date: "2011-02-21"
tags: 
  - "php"
  - "technology"
  - "tips"
  - "js"
  - "performance"
---

One of the golden rules when we want a high performance web site is minimize the HTTP requests. Normally we have several JavaScript files within our projects. It’s a very good practice to combine all our JavaScript files into an only one file. We can do it manually. Not a hard work. We only need to copy and paste the source files into our single js file. There's even tools to do it. You can have look to [Yslow](https://addons.mozilla.org/en-US/firefox/addon/yslow/) (if you don’t know it yet). That’s a good solution if your project is finished. But if your project is alive and you are changing it, it’s helpful to spare your JavaScript files between several files. It’s good to organize them (at least for me). So we need to choose between high performance and development comfort. Because of that I like to use the simple script I’m going to show now. Let’s start.

The idea is the following one. Normally I have all js files into a folder called js (original isn’t?). I also have a development server and a production one (really original again, isn’t it?). When I’m developing my application I like to have my js files separated and in a human readable way (I’m human), but in production I want to combine them and even minimized and gziped to improve the performance.

The script I have is a simple script that combines all the JavaScript files, minimizes and gzips.

```php
//js.php
require 'jsmin.php';
 
function checkCanGzip(){
    if (array_key_exists('HTTP_ACCEPT_ENCODING', $_SERVER)) {
        if (strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'gzip') !== false) return "gzip";
        if (strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'x-gzip') !== false) return "x-gzip";
    }
    return false;
}
 
function gzDocOut($contents, $level=6){
    $return = array();
    $return[] = "\x1f\x8b\x08\x00\x00\x00\x00\x00";
    $size = strlen($contents);
    $crc = crc32($contents);
    $contents = gzcompress($contents,$level);
    $contents = substr($contents, 0, strlen($contents) - 4);
    $return[] = $contents;
    $return[] = pack('V',$crc);
    $return[] = pack('V',$size);
    return implode(null, $return);
}
 
$ite = new RecursiveDirectoryIterator(dirname(__FILE__));
foreach(new RecursiveIteratorIterator($ite) as $file => $fileInfo) {
    $extension = strtolower(pathinfo($file, PATHINFO_EXTENSION));
    if ($extension == 'js') {
        $f = $fileInfo->openFile('r');
        $fdata = "";
        while ( ! $f->eof()) {
            $fdata .= $f->fgets();
        }
        $buffer[] = $fdata;
    }
}
 
$output = JSMin::minify(implode(";\n", $buffer));
 
header("Content-type: application/x-javascript; charset: UTF-8");
$forceGz    = filter_input(INPUT_GET, 'gz', FILTER_SANITIZE_STRING);
$forcePlain = filter_input(INPUT_GET, 'plain', FILTER_SANITIZE_STRING);
 
$encoding = checkCanGzip();
if ($forceGz) {
    header("Content-Encoding: {$encoding}");
    echo gzDocOut($output);
} elseif ($forcePlain) {
    echo $output;
} else {
    if ($encoding){
        header("Content-Encoding: {$encoding}");
        echo GzDocOut($output);
    } else {
        echo $output;
    }
}
```

As you can see the script checks recursively all the js files inside one folder, combine them and also use [jsmin](https://github.com/rgrove/jsmin-php/) library for PHP to improve the download time in the browser.

It’s very easy now when we’re building our HTML file switch from one js file to another. Here you can see an example with Smarty template engine:

```html
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title></title>
    </head>
    <body>
        Hello World
{if $dev}
        <script src="js1.js" type="text/javascript"></script>
        <script src="js2.js" type="text/javascript"></script>
        <script src="xxx/js1.js" type="text/javascript"></script>
{else}
        <script src="js.php" type="text/javascript"></script>
{/if}
    </body>
</html>
```

Yes. I know. There’s a problem with this solution. Maybe we’ve improved the client side performance reducing the number of HTTP requests but, what about our server side performance? We’ve changed from serving static js files to dinamic PHP file. Now our server’s CPU will work more. Another great hight performance golden rule is to place static files into a server dedicated to serve static files (without PHP support). Whit this golden rule we help to the browser to perform multiple downloads and also we reduce the use of CPU (static server will not instance any PHP session).

So a better solution is the offline generation of the static js file when we deploy the application. I do it with a simple curl at command line.

```commandline
curl http://nov/js/js.php -o jsfull.minified.js

```

So the smarty template will change to

```html
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title></title>
    </head>
    <body>
        Hello World
{if $dev}
        <script src="js1.js" type="text/javascript"></script>
        <script src="js2.js" type="text/javascript"></script>
        <script src="xxx/js1.js" type="text/javascript"></script>
{else}
        <script src="jsfull.minified.js" type="text/javascript"></script>
{/if}
    </body>
</html>
```

It’s also a good solution put a prefix in our JavaScript file and change it each time we build it (easy to automate) to ensure the cache renew.

```html
<script src="jsfull.minified.20110216.js" type="text/javascript"></script>
```

And yes we can use the same trick with our css files.
