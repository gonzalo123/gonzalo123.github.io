---
layout: post
title: "My development tips"
date: "2010-08-23"
tags: 
  - "php"
categories: 
  - "php"
---

Another unsorted list of ideas this time about coding tips.

## code = mass

The source code isn’t an abstract element. OK it’s not as touchable as apples or bricks but it’s somehow under physical laws too. You must write as less code as you can to meet your requirements. Big code means big mass and if you have a big mass you will need more energy to change whatever you need. I can stop a car toy with my hand but I need something more to stop a real train with the same speed. Less code means less failure points and it's easier to manage. Some people claim proudly they have written a library with thousands of code lines. Nobody pay us for writing lines of code. They pay us for the solutions. The lines of code that we write is our problem. So be as minimalistic as you can. The problem is that writing less code is more complicated than write more. We need to think more to write less code.

## DRY (Don’t repeat yourself)

Original, isn’t it?.  Copy & paste is evil. Never use it in your projects. Spread the same code among different files means that you will need to remember where are those pieces of code when a bug appear (they appear, believe me). There’re a lot of techniques to avoid copy & paste source code. Use them. There isn’t any excuse to it.

## Coding standards

The source code is one of your deliverables as developer. You must care about it. Adopt a coding standard. Chose one and use it. If you work with a team take this decision with your team, but don’t try to create one standard by your own. It’s a big job. If you use one of the existing coding standards (let’s say Zend Framework’s one) you will find tools to validate your code against it and the most common IDEs and text editors will be able to use it too. In other way if you create a new one you will need to develop those plug-ins and tools and even you will need to document it if you want to show it to another developer. To many work that nobody will pay for it only because you dismiss an existing one.

## Revision control.

Mandatory. Even for small and personal projects. I like Mercurial. it’s easy to use and easy to install. Anyway there are others similar like git or bazzar. Or if you are old school, CVS and Subversion. When I start a project, after create an empty folder with the name of the project I always execute “hg init” to create my repository. Commit you code often. Every time you make something important commit your work. Don’t wait until your project is finished. Revision control is a great backup system too. When something wrong happens like accidentally delete of a folder or something similar you can recover it easily without any problem. Also with a simple hg push (or similar command with another software different from mercurial) you will save your project into another place. If you don’t know to use any revision control system aside time for it. Believe me. You will not need too many time to learn how to use it. At least a basic usage.

## Text editor

Is your main tool as a developer. It’s like the hammer for a carpenter. You must train yourself in the use of it. A good skill in the use of a text editor or IDE will give great benefits. If you think in term of ROI, the investment you use in your IDE will return sooner or later. Some of them are not trivial. For example Eclipse needs time to learn how to configure it. You must skill yourself into the IDE if you don’t want to lose your time developing (remember time = money). I’ve seen people programming PHP with notepad. You need at least syntax highlight, syntax error detection, auto-completion, auto-completion with code introspection and debugging. Don’t waste your time.
