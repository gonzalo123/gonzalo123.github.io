---
layout: post
title: "Easy trick to move cache files to RAM without coding a PHP line."
date: "2010-02-13"
tags: 
  - "php"
categories: 
  - "php"
---

Some year ago I faced to a problem to increase the performance of a web application. This application used [Smarty](http://www.smarty.net/) as template engine. Smarty 'compiles' the templates (the tpl files) into tpl.php files.

As I saw in the server logs almost all I/O reads were to the tpl.php files. Those files where with other cache files in a cache dir on the file system. Nowadays template engine allows to use some RAM backends like memcache or similar. But I don't wanted neither touch any line of code nor change the behaviour of the template engine. So I decided to do an easy trick.

Server was a linux box. Linux systems has [/dev/shm](http://www.cyberciti.biz/tips/what-is-devshm-and-its-practical-usage.html). shm is a temp file system on shared memory. Normally it's mounted with a percent of our RAM memory and we can use it if we want to increase the performance or our application. I/O operation over RAM are faster than hard disks.

We only must take care about one thing. When we power down the host all data of our tempfs will be lost. So it's perfect to save cache files. I only need to delete cache dir and create a symbolic link to /dev/shm

```commandline
rm -Rf cache/
ln -s /dev/shm cache/
```

And that's it. All cache reads will be done now in RAM memory instead of  disk.

As I said before we must take care /dev/shm flushes when we power down the server. So if our cache system uses subfolder, we must take care the application will not assume the folders are already created. If we have a tree in our cache directory and we don't want to change a line in our application we can create the tree in a script and put the script in the start-up of the server.

Easy, fast and clean. I like it.
