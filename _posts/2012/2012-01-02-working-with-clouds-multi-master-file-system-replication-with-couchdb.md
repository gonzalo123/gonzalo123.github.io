---
layout: post
title: "Working with clouds. Multi-master file-system replication with CouchDB"
date: "2012-01-02"
tags: 
  - "cloud"
  - "couchdb"
  - "php"
  - "technology"
---

When we want to work with a cloud/cluster one of the most common problems is the file-system. It's mandatory to be able to scale horizontally (scale out). We need to share the same file-system between our nodes. We can do it with a file server ([samba](http://en.wikipedia.org/wiki/Samba_(software)) for example), but this solution inserts a huge bottleneck into our application. There's different distributed filesystems such as Apache [Hadoop](http://hadoop.apache.org/) (inspired by Google’s MapReduce and Google File System). In this post we’re going to build a really simple distributed storage system based on NoSql. Let’s start.

NoSql (aka one of our last hypes) databases normally allow to store large files. MongoDB for example has [GridFS](http://www.mongodb.org/display/DOCS/GridFS), But in this post we’re going to do it using [CouchDB](http://couchdb.apache.org/). With CouchDB we can attach documents within our database as simple attachments, just like email.

The api of CouchDB to upload an attachment is very simple. It’s a pure REST [api](http://wiki.apache.org/couchdb/HTTP_Document_API#Attachments).

In order to create the http connections directly with curl commands we can use libraries to automate this process. For example we can use a simple library shown in a previous [post](http://gonzalo123.wordpress.com/2010/08/30/using-couchdb-as-filesystem-with-php/). If we inspect the [code](https://github.com/gonzalo123/nov-couchdb/blob/master/Nov_Http.php#L260) we will see that we’re creating a PUT request to store the file in our couchDB database.

Another cool thing we can do with PHP is to create a stream-wrapper to use standard filesystem functions for read/write/delete we can see a post about this [here](http://gonzalo123.wordpress.com/2010/09/06/using-a-stream-wrapper-to-access-couchdb-attachments-with-php/).

As we can see is very easy to use couchdb as filesystem. but we also can replicate our couchDB databases. Imagine that we have tho couchDB servers (host1 and host2). Each host has one database called fs. If we run the following command:

```commandline
curl -X POST -H "Content-Type: application/json" http://host1:5984/_replicate -d '{"source":"cmr","target":"http://host2:5984/fs","continuous":true}'

```

Our database will be replicated from host1 to host2 in a continuous mode. That’s means everytime we create/delete anything in host1, couchDB will replicate it to host2. A simple master-slave replica.

Now if we execute the same command in host2:

```commandline
curl -X POST -H "Content-Type: application/json" http://host2:5984/_replicate -d '{"source":"cmr","target":"http://host1:5984/fs","continuous":true}'
```

We have a multi-master replica system, cheap and easy to implement. As we can see we only need to install couchDB in each node, activate the replica and that’s all. Pretty straightforward, isn't it?
