---
layout: post
title: "Detecting errors and bottle necks in PHP."
date: "2010-06-22"
tags: 
  - "php"
categories: 
  - "php"
---

In this post I want to make a short list of different ways to detect errors and bottle necks in our PHP projects. It’s an informal list of recommendations and best practises useful at least for me

### Indent code.

Python has a great characteristic. Code blocks are defined with indentation. Not with any especial character like other program languages. That’s means if you don’t want syntax errors and you want to execute your program, your script must be properly indented. That’s is a great practise., even in program languages not as restrictive as python with code indention. For example when I need to check a piece of code with a any kind of error, if it isn’t correctly indented I start to indent it. I know it isn’t necessary but for me at least it helps me to understand the code (if I'm not the original developer of the code) and quickly find errors like wrong if clauses and another things like that.

### Analyze code and check syntax

You must use a modern text editor or IDE with at least check syntax and also code analyze tools. That’s mandatory. Chose what you want but use one. Zend Studio is a great tool but you also can do it with another editors, even with vi. You can programing PHP with notepad but it’s definitely a waste of time. You will use significant less time programming with modern IDE. I like Zend Studio code Analyzer. It helps you with errors easy to implement and difficult to find like :

```php
// error 1
if ($a=1) {
 
// error 2
$variable= 2;
if ($Variable == 2) {
...
}
```

I like to fix every warning even if those warnings are recommendations and not in fact a problem.

### Check indentation level.

Even with a perfect indentation big indentation level is a clear signal of “[code smell](http://en.wikipedia.org/wiki/Code_smell)”. Take care of it. If you see a big indentation level you must ask yourself: Is it really the way to do it?

### Loops.

If you want to check performance issues and you don’t have too many time,focus yourself finding loops and analyze them. If there is a performance problem is very probably to discover your problem inside a loop. One slow operation executed one time is a problem. But if this operation is executed 1000 times inside a loop is definitely a big problem. OK it’s the same problem but the impact is 1000 times bigger.

### Copy-paste code

Avoid them like a plague. It can be helpfully and attractive but the use of the same piece of code distributed among a set of source code files will give you definitely a headache in the future, Be sure about it. We have a lot techniques in to avoid copy and paste code.
