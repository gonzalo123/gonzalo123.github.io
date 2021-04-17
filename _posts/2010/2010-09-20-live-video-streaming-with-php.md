---
layout: post
title: "Live video streaming with PHP"
date: "2010-09-20"
tags: 
  - "php"
categories: 
  - "php"
---

Picture this. We want to stream live video. In fact don’t need PHP. We only need a flash player (or HTML5) and our live feed. The problem appear when we need to offer some kind of security. For example we want to show videos only to registered users based on our authentication system. Imagine we’re using sessions for validate users. That’s means we cannot put the media in a public folder and point our media player to those files. We can obfuscate the file name but it’ll remain public. In this small tutorial We’re going to see how to implement it with PHP. Let’s start.

First we are going to forget for a while video files. We are going to think in pictures.

If we don’t need to protect the file we can place the image in public folder:

```php
<image href=’/path/to/image.png’ alt=’image’ width=’10’ height=’10’/>
```

But if we want to protect the picture we cannot place it on a public folder. We need to place it on a server folder and flush it with a code like this:

```php
// here you can chech your sessions
header("Content-Type: image/jpeg");
readfile(‘/path/of/image’);
```

That’s easy. Isn’t it?. Video files are similar than pictures. We cannot use image tag but the behaviour is the same. With HTML5 we can show them with HTML tags and without HTML5 we need to use a client viewer (generally a flash viewer).

We can use the same technique than the previous example with video files but the problem now is the size of the file. Live video feeds are never ends. If we use the previous code our server must read the live feed forever, and our browser will be waiting forever too. We can easily change this behaviour. Instead to read the entire video file and flush it to the browser we can read small pieces of the file and flush them without waiting until the end of the file.

```php
<?php
// here you can chech your sessions
 
function flush_buffers(){
    ob_end_flush();
    ob_flush();
    flush();
    ob_start();
}
 
$filename = "/path/of/fideofeed.flv";
if (is_file($filename)) {
    header('Content-Type: video/x-flv');
    header("Content-Disposition: attachment; filename=video.flv");
    $fd = fopen($filename, "r");
    while(!feof($fd)) {
        echo fread($fd, 1024 * 5);
        flush_buffers();
        }
    fclose ($fd);
    exit();
}
```

If we want to stream static video files we also need to take into account the start of the video. When someone want to move across the video going forward and backward within the video client, our server script will retrieve a variable with the start time. That's means we need to use this start time to open the file from this byte instead from the beggining.
