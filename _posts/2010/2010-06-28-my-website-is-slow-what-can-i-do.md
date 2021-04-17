---
layout: post
title: "My website is slow. What can I do?"
date: "2010-06-28"
tags: 
  - "php"
categories: 
  - "php"
---

You are working with a website. The website works. All is perfect but your clients told you it’s very slow. You must face the problem and improve the behaviour of the web site. But remember all site works properly. There isn’t any errors. The common way to resolve problems (understand the problem, reproduce the problem in test environment, solve the problem) doesn’t fit in this scenario. What can we do? I want to give some recommendations to improve performance problems in this post. Let’s go

## Don’t assume anything

It’s very typical in our work to assume what is the problem when someone gives us an issue. Normally the user is not a technician. He suffer a problem and tell us the symptoms. For example some people from procurement office call you telling the application doesn’t work. You assume he is speaking about the application he use every day doesn’t work. You can go to check the server. It’s OK. Server logs OK. What happens? Finally you discover there is a network problem. Not a problem in the application. but the effects to the users are the same.

When you face a performance issue don’t assume anything. First of all you must debrief the user to take a picture of the problem. Forget the solution just now. In this phase you only need to collect the information and perform an analysis later. If it possible speak and go to his office and see the problem with him. I remember a performance problem some time ago. The user had serious problems with the application. I tested the application and I didn't find anything wrong. But the problem persists and she was the main user of the application. I went to her office and I discover the real problem. The application was slow. But the screen-saver was slow too. Spreadsheet was slow. In fact all was slow there. The problem was the RAM memory of the PC. More RAM and magically all applications become faster.

## Measure the problem

If something is slow you must check times. But take care about it. Wrong measures can make you waste your time trying to solve the problem. A typical mistake is check times only at server-side. For example you start a timer when the script start and finish it when it ends. Imagine you have 1 second. It can be improved but, is the end-user complaining for a performance issue with 1 second of response time? Probably not. You can start to improve your server-side code. You spend some time coding and you turn from 1 second to 0.1 second. You are very proud of your improvements, and you tell to the user the problem is solved. But the user is not agree with you. The problem persists. You’ve been working on a different problem. A problem indeed but different one than user claims. Why? Because you’ve assumed the problem was in server code and your time measurements have been done in a wrong scenario.It’s quite probably the problem in in client site. If you take a look into firebug’s net tab you can realize server-side part (e.g. php ones) normally is the first one but is not the only one. Even it’s not the longest part. It can be a short percent of full-page load and render time. If you want to acheive significant success within your performance problem you must attack directly to the main bottle neck (and detect them before of course). If you want to learn a lot about client side performance, please pick up [Steve Soulders](http://stevesouders.com/)’s “High performance web sites” book. You can also read the another Steve’s book “Even faster web sites” but first one is definitely a _must read_ book for people who work in this area. You can also see many conferences of great Steve Soulders in youtuve. Do it. He is a great guru in this area and also a good speaker. After watching his conferences you will have  the desire to drink a beer with him. Probably working on “High performance web sites”’s recommendations you will achieve a significant results, following a really simple rules.

## Cache it

I know I not very original giving recommendations about caching in this post but caching is very important. There is a lot theory about caching. You must cache all as you can. But don’t do it like mad. You will get a cache nightmare if you don’ have a good caching plan. You must define the storage, the ttl (time to live) and what is going to be cached and what not.  A wrong caching politics can jeopardize a project but a good caching ones will improve the performance.

## Do it offline

Doing all online is cool. The user will get the fresh results when he clicks a button but what happen if the action takes too many time?. Imagine you have a button that sends ten emails every time the user clicks on it. In normal situations the operation is fast enough but what happens if mail server has a big load, or even it’s down. Your application will freeze and your user will become angry. Think moving operations to background. There are great tools like [gearman](http://gearman.org/) to perform those kind of work. Transform your button from: user clicks, mail one is sent, mail 2, ... mail 10 is sent, OK  to: use clicks, new task in our job server, OK. Now true doesn't mean the ten emails have been sent. Now means they will be send. Balance the possibility of this new behaviour. I realize sometimes it isn't possible but it is viable in other cases.Imagine you have an important report that uses a complex SQL to extract information from database. This SQL uses several tables to met user expectations. Is it mandatory to perform always the query to get the results? Think in the possibility of creating some statistic tables to collect old information (non changed ones, such as old months or years). Take a snapshot offline of your real-time data and perform queries over those snapshots instead of real information. Sometimes this technique is not possible but if it is available you can achieve important time benefits within your database queries.

## Database Connections.

Working with relational databases the creation of database connection is a slow operation. Take care about it. It’s a good practise to put a counter in your script to show you how many connections you create in your script, how many queries and how many results gives your queries to your application. You can realize unpleasant problems with this simple log. If your application perform more than one connection to the same database within the same execution script it’s very sure you are doing something wrong. Use always lazy connections to the database. I also have seen scripts that connects to the database without doing any operation. That's means you are wasting time connection to the database. Connect only when you really are going to use it. Not always in the beginning of the script as general rule.Check the sql you are using. If for example you are always doing the same query every click on the site to check some kind user information a red light must appear in your mind with a flashing box with the text: cache it! inside.

Trace long query and analyze them into the database. Check indexes and execution plans. This normally is a great bottle neck in the web applications.

## Debug flags

Use debug flags to measure the problem. Firebug in combination with FirePHP are a great team to help us in our work. But don’t forget to turn those flag off in the production server. To many debug information collectors active in our production servers will slow down or application with unnecessary actions
