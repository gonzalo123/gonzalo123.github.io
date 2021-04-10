---
layout: post
title: "Watermarks in our images with PHP and Gearman"
date: "2011-03-07"
tags: 
  - "php"
  - "technology"
  - "gd"
  - "gearman"
---

In my last [post](http://gonzalo123.wordpress.com/2011/02/28/watermarks-in-our-images-with-php-and-mod_rewrite/) I've tried to explain how to add a dynamic watermarks in our images using PHP and the GD library. Someone told me in a [comment](http://gonzalo123.wordpress.com/2011/02/28/watermarks-in-our-images-with-php-and-mod_rewrite/#comment-996) that he has used something similar, but he had to disable it because of the use of huge resources. Probably the use of the solution of the previous post doesn't scale well. If you have a hight traffic site your memory and CPU usage will increase a lot because of the image transformation (especially if you work with big images). It'd be better if you can generate the watermarks offline, but if is mandatory to create them dynamically (for example we need to place the current timestamp), there are other solutions.

In this second solution I will use a [gearman](http://gearman.org/) worker to generate the watermarks. The benefits of [gearman](http://gearman.org/?id=gearman_php_extension) is the possibility of use a [pool](http://www.php.net/manual/en/gearmanclient.addserver.php) of workers. We can add/remove workers if our application scales. Those workers can be placed even at different hosts, and we can swap easily from one configuration to another. Imagine we have an application that uses a single worker at the same host of the webserver. Maybe it's enough for a small site, but suddenly we increase our users. We can add new workers to our host. But if our single host is not enough, we can rent new host/hosts (with amazon for example) and our application will adapt easily to the new scenario. Gearman allows an easy way to scale out our applications.

Let's start:

Now our main script instead of doing the hard work, will be a gearman client

```php
<?php 
$filename = "/path/to/img.jpg";
$footer = date('d/m/Y H:i:s');
 
$gmclient = new GearmanClient();
$gmclient->addServer();
 
$handle = $gmclient->do("watermark", json_encode(array($filename, $footer)));
 
if ($gmclient->returnCode() != GEARMAN_SUCCESS){
    echo "Ups something wrong happen";
} else {
    header( 'Content-Type: image/jpeg' );
    echo $handle;
}
```

And our worker will do the hard work.

```php
<?php 
$gmw = new GearmanWorker(); 
$gmw->addServer();
$gmw->addFunction("watermark", function($job) {
 
    $workload = $job->workload();
    $workload_size = $job->workloadSize();
 
    list($filename, $footer) = json_decode($workload);
 
    $footerSize = 15;
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
 
    ob_start();
    ob_implicit_flush(0);
    imagepng($im);
    $img=ob_get_contents();
    ob_end_clean();
 
    return $img;
});
while(1) {
  $gmw->work();
}
```

We must remember to start our worker/workers within our server.

```commandline
php /path/to/worker/worker.watermark.php
```

What do you think with this solution?
