---
layout: post
title: "Digressing about working with old code, refactoring and scope creep."
date: "2010-08-16"
tags: 
  - "php"
categories: 
  - "php"
---

Maybe there’re people who write code without bugs and never need to change one line of their source code. That’s not me. I try to improve my coding skill for the years. That means sometimes I need to face against my old code and if need to rewrite it probably I will do in a different way. I always feel the desire to refactor it. Refactor code is a good thing but is not always viable.Let’s show an example. Nowadays I’ve embraced [Zend coding standard](http://framework.zend.com/manual/en/coding-standard.html). For instance one not camelized variable creeps me out. But in my early days as a PHP developer (almost ten years ago) I didn’t take care about any coding standards. If code worked, it was OK. Now I realized that’s not enough. Code must work. Indeed!. But it must be ready to be changed, adapted to new needs and meet new requirements, and another developer must be able to understand it. Last days before holidays I’ve been working adapting a project to [PHP5.3](http://es2.php.net/migration53). It’s not a hard job. Some functions have disappeared, now we only have one kind of regular expressions and a couple of things more. I need to check a lot of lines of code. Sometimes I blush myself when I see the code I wrote five or even eight years ago. I want to change a lot of things. Refactor again and again but I must remember that’s not in the scope of the project. The objective of the project is clear: The application must work in a server with PHP5.3. If I start to refactor code the project deviates from project's main scope. That’s a clear [scope creep](http://en.wikipedia.org/wiki/Scope_creep)’s signal.

According with theory the main signals of scope creep are:

- More work has been done than required
- Product features don’t match the specifications.

Scope creep one of the most important problems in real projects. It’s very difficult to explain to someone that you will delay your project because you have decided to refactor some pieces of code (code that worked). That's means your project will be delayed because you’ve decided to improve something that nobody ask for it. If you are on schedule maybe you can assume some refactor but think this kind of work is only viable if the [triple constraint model](http://en.wikipedia.org/wiki/Project_management#Project_Management_Triangle) (aka Scope + Budget + Schedule) keep unaltered. You must assess if those unsolicited changes have any impact in the project. Projects must end at some point. At some point you will need to submit the project's deliverables, put your feet up and relax. But the temptation is big. I know.
