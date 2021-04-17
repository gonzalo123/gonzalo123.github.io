---
layout: post
title: "Clustering PHP applications. Tips and hints"
date: "2010-07-27"
tags: 
  - "php"
  - "couchDB"
categories: 
  - "php"
  - "couchDB"
---

Sometimes a web server and a database is fair enough to meet our project requirements. But if the project scales we probably need to think in a clustered solution. This post is an attempt at being an unsorted list of ideas working with clustered PHP applications. Maybe more than a list of ideas is a list of problems that you will face when swapping from a standalone server to a clustered server.

## Source code

If you’re going to spread your code among different servers, one important fact is the source code must be always the same. You cannot assume differences in the source code. If you need to mark something related to the node, use environ variables and point your code to those environ variables instead of using different source code in each node. Perhaps changing the code is the fastest way to do the things initially but you will mess up your deploy system or at least turn it into a nightmare.

## Databases

Here you are the biggest problem. Clustering databases is a hard job. All Databases normally have one kind or another of replica system but replication as way to solve problems can backfire on you. The easiest way of database replication is a master slave replica system. Inserts in one node and selects in all. This scenario can be easy to implement and not very hard to maintain but maybe you need a multi-master replica system (inserts in all) and this can turn your database administration into a nightmare. One solution is use [noSql](http://en.wikipedia.org/wiki/NoSQL) databases. They have a simple and useful multi-master replica system. My recommendation is use a noSql database for those kind of information we need to have distributed in all nodes. I know it isn’t always straightforward but all the work done in this direction will be will be worthwhile in our outcomes. In fact noSQL' main purpose is data scalability but also there are people who thinks noSql databases are some kind of hype actually. Who is right?

## File-systems

Problems again. Imagine you need to write a simple log system in your application. fopen, fwrite, a few PHP lines and your log is ready. But: Where is it located?. Probably on your local filesystem. If you want to place your application on a distributed cluster, you cannot use those simple functions. You must consider using a distributed filesystem. Maybe a simple rsync would work. Maybe a single fileserver mounted on every node of your cluster. You need to balance between them and choose the solution that fits to your requirements. One smart and simple solution is benefit from CouchDB’s attachments in addition to its replication system. (can you feel I like [CouchDb](http://couchdb.apache.org/)?)

## Deploy

That’s a mandatory point. You must have an effective deploy system. You can’t rely on you to deploy the code on the among the cluster by hand. This work must be automated. [PHING](http://phing.info/trac/), [rsync](http://www.samba.org/rsync/), pull/push actions in a distributed revision control (e.g. [mercurial](http://mercurial.selenic.com/)) or maybe a simple bash script can be enough. But never do it by hand.

## Authentication and Authorization

If your application is protected with any authentication mechanism you must ensure all nodes will be able to authenticate against the same database. A good pattern is to work with an external authentication mechanism such as [OAuth](http://oauth.net/) in a separate server. But if it’s not possible for you must consider where is located user/password database. CouchDB is good solution.

After your user is authenticated you need no check the authorization. OAuth servers can help you on authentication but your application must deal with authorization (aka user roles and role groups) anyway. My best choice here is CouchDb but you can use relational databases too. As I said in Databases section I avoid multi-master Database Replication with relational databases like the plage, but if you need to use relational replication in the authorization database you can use master-slave replica and update all role and authorizations in the master database.

## Sessions, Logs and Cache files

Witch session handler do you use in your application?

File based (go to filesystem section of this post),  Database (go to database section of this post).

The same with cache and logs. With cache files you can consider keep it locally. If your cache system works unattended you can maintain the cache in local node instead to replicate it among the cluster. You can save bandwidth.
