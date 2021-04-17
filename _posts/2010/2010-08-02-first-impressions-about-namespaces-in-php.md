---
layout: post
title: "First impressions about namespaces in PHP"
date: "2010-08-02"
tags: 
  - "php"
categories: 
  - "php"
---

I've been working wih my first project using namenspaces in PHP. As a PHP developer I have been suffering the lack of a program language without namespaces for years. But now it’s over. Finally we have namespaces in PHP since PHP 5.3. My early impressions wasn’t good. I had a problem. I really liked PEAR naming conventions. I know that leads to those ugly long class names, but I felt very comfortable with them. I even started to write a post in my blog called “Things I don’t like in PHP. Namespaces”. But I decided not to publish it since have been working a bit seriously with them. Now I have use them and my impressions have been changed. I’ve embraced them and they are not so bad. They have the same advantages than PEAR naming conventions and they give us some extra benefits further. [Here](http://blog.nickbelhomme.com/php/php-5-3-3-namespaces_251) we have a great article about namenspaces in PHP5.3

The only doubt I have is: Are they really better than classical naming conventions? I make myself the question because as well as in classical naming convention the Classes are fully defined, with namespaces we must take care about aliases, ‘use’ statements and the scope. When I say 'fully defined' it means that in a glance to the class name we known exactly the location of the class in the filesystem.. We pay a tax for it. Our class name become bigger. Namespaces come to help us but they add to our code a bit of complexity. I feel really comfortable with classical naming conventions but namespaces are cool. Are they some kind of hype? I don’t think so but I am not 100% convinced. Yet. What do you think?
