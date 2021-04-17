---
layout: post
title: "Sessions, comet and PHP"
date: "2010-04-13"
tags: 
  - "php"
categories: 
  - "php"
---

I faced this problem when I was developing a comet server but it can happen with any script that needs too many time. I have done a comet process. This process is very simple. Basically is a PHP script who looks the modification date of a file. When it changes, the script ends but if nothing happens the script ends after 30 seconds and start again (with a JavaScript loop). The script worked perfectly on sandbox. In production also worked (brilliant isn't it?). But the problem appears when I open other tabs in the browser. The application became slow. Very slow. Every click, even really simple operations turned into unusable operations. I realized this behavior appear only when comet was enabled.

A small skeleton of comet server (the code ):

```php
for ($i=0; $i<10; $i++) {
    if (checkSomething()) {
        echo getData();
        flush();
    }
    sleep(1);
}
```

The problem was in the authentication. The comet server uses session for authentication. The session are stored as files. The system worked perfectly but I realized I didn't use session\_write\_close. That's means server open session file and frees it when the script ends. Normally script takes one second or less but the comet server may take 20-30 seconds.

```php
auth();
for ($i=0; $i<10; $i++) {
    if (checkSomething()) {
        echo getData();
        flush();
    }
    sleep(1);
}
```

In this case the solution was easy. The auth process was only in the beginning of the script so I only need to use session\_write\_close after authentication process. With this simple command the server doesn't lock user sessions and I can ope as many tabs as I need.

```php
auth();
session_write_close(); // this command is on auth function but I put it here for legibility purposes
 
for ($i=0; $i<10; $i++) {
    if (checkSomething()) {
        echo getData();
        flush();
    }
    sleep(1);
}
```

There are other storage to session files instead of filesystem (it's the default one). In relational database, non-relational database and even in memory with mm
