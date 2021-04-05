---
layout: post
title: "Calling Silex backend from command line. Creating SAAS command line tools"
date: "2016-01-25"
tags: 
  - "php"
  - "technology"
  - "cli"
  - "curl"
  - "silex"
---

Sometimes we need to create command line tools. We can build those tools using different technologies. In Symfony world there's [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html). I feel very confortable using it. But if we want to distribute our tool we will need to face with one "problem". User'll need to have PHP installed. It sounds trivial but it isn't installed in every computer. We can use nodeJs to build our tool. Nowadays nodeJs is a de-facto standard but we still have the problem. Another "problem" is how to distribute new version of our tool. Problems everywhere.

Software as a service tools are great. We can build a service (a web based service for example) and we can even monetize our service with one kind of paid-plan or another. With our SAAS we don't need to worry about redistribute our software within each release. But, what happens when our service is a command line one?

Imagine for example that we're going to build one service to convert text to uppercase (I thing this idea will become me rich, indeed :)

We can create one simple Silex example to convert to upper case strings: 

```php
<?php
include __DIR__ . "/../vendor/autoload.php";
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application();
$app->post("/", function (Request $request) {
    return strtoupper($request->getContent());
});
$app->run();
```

And now we only to call this service from the command line. We can use curl for example and convert one file content to upper case:

```commandline
cat myfile.txt | curl -d @- localhost:8080 > MYFILE.txt
```

You can see the example in my github account [here](https://github.com/gonzalo123/uppersaas)
