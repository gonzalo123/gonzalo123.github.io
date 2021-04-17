---
layout: post
title: "Keep our PostgreSQL databases syncronized with PHP. Database version control."
date: "2010-12-13"
tags: 
  - "php"
  - "postgreSQL"
categories: 
  - "php"
  - "postgreSQL"
---

Last October I attended to PHP [Barcelona](http://phpconference.es/) 2010. One of the talk I really wanted to see was “[Database version control without pain](http://www.slideshare.net/harrieverveer/database-version-control-without-pain-the-php-barcelona-version)”. In this talk [Harrie Verveer](http://twitter.com/#!/harrieverveer) showed us different tools to keep synchronized our databases. This is a really common problem working with databases. Source code is well covered with source control management tools. I normally use mercurial (hg) but with git, bazaar or svn we can cover all our needs within source code. We create source code at development server and push the changes to production. It’s really easy to keep synchronized all our code. But with databases it’s different. Maybe Oracle's people are few steps above the rest and they have something similar than source code control for database schema's natively in the database. If you add a new column to a table, your schema will be transformed into a new version. Oracle database is great. It has incredible features and really good performance, if you can afford the fee. In this post I will try to solve this problem with [PostgreSQL](http://www.postgresql.org/). A really good database similar than Oracle and open source and free (as _"free beer" also_)

It's a recurrent problem working with databases. We create database objects (tables, views, ..) in the development server and when our application is ready to go live we push the changes to production server. If we are smart developers we save all database scripts in a file and when we deploy them to production we execute the script. There are tools to do it like [dbdeploy](http://dbdeploy.com/) or even [phing](http://phing.info/) (You must have a look to Harrie's [presentation](http://conference.phpnw.org.uk/phpnw10/2010/12/03/phpnw10-harrie-verveer-database-version-control-without-pain/) to see all the possibilities). The problem is that we must be very strict and we must create the script even when we alter a column in a bug fix. It's a hard work and it will be worthwhile but we must be mindful it's really easy and fast to alter a column with a IDE (pgadmin for example) and very easy to forget to save the diff file. If you work alone maybe you can afford it but if you work within a team is difficult to ensure everybody update the diff file by hand. The purpose of [this library](https://code.google.com/p/pgdbsync/) is to create the diff scrip automatically according to the current state of the database. Lets's start.

As Harrie said there isn't a silver bullet solution (I really went to his talk finding the silver bullet :) ). My first attempt was to use Doctrine2. Doctrine2 has a great tool called [dbal](http://www.doctrine-project.org/projects/dbal). I tried to use it but I faced with a big problem, at least for me. I like to organize my database objects (tables, views, ...) within schemas. With Doctrine2’s dbal I have problems and it doesn't work as I think it must work. It assumes all objects are in the default schema and it doesn’t add “\[schema\].” to the information schema queries.  Maybe is my fault but If want to use dbal I need to hack the code. Because of that I started to write [pgdbsync](http://code.google.com/p/pgdbsync/).

It would be cool to create a multi-database library to synchronize our databases. But it’s a huge work for doing alone so I've focused only on PostgreSQL, because is the database I mainly use in my daily work.

The Idea is to create a command line tool to create the need script to keep synchronized two (or more databases). We must take care we're speaking about database's schema. Not database's data.

First I create a ini [file](http://code.google.com/p/pgdbsync/source/browse/conf.ini) with the configuration of each database connection. The user we use to connect to the database must have access to data dictionary in our PostgreSQL database.

```ini
[devel]
TYPE = pgsql
HOST = development
PORT = 5432
DBNAME = developement
USER = user
PASSWORD = password
 
[prod1]
TYPE = pgsql
HOST = production
PORT = 5432
DBNAME = prod1
USER = user
PASSWORD = password
```

I've created a simple CLI script to use the library with [getopt](http://php.net/manual/es/function.getopt.php). You can see the script [here](http://code.google.com/p/pgdbsync/source/browse/pgdbsync). The usage of the script is very simple. I have implemented three main actions: **diff**, **summary** and **run**.

- **diff**: Calculates the needed script to keep synchronized the databases. Prints the script on the screen but it doesn't executes anything.
- **summary**: Shows a summary the differences. Not the full script. Useful to see the differences in a glance.
- **run**: Calculates the diff function and executes it.

Let’s start. First we start with an empty database in Development and Production servers with a schema WEB in both sides (schema creation's coverage is not supported by pgdbsync). Then we create a few objects on the development server.

```postgresql
-- TABLE TEST
CREATE TABLE WEB.TEST (
  TEST_NAME CHAR(30) NOT NULL,
  TEST_ID INTEGER NOT NULL,
  TEST_DATE TIMESTAMP NOT NULL
)
TABLESPACE WEB;
ALTER TABLE WEB.TEST ADD CONSTRAINT PK_TEST PRIMARY KEY (TEST_ID);
 
-- VIEW on TEST
CREATE VIEW WEB.testview(
  TEST_NAME,
  TEST_ID,
  TEST_DATE
) AS
SELECT *
FROM WEB.TEST
WHERE TEST_NAME LIKE 't%';
 
-- Function hello
CREATE OR REPLACE FUNCTION WEB.hello(item character varying)
  RETURNS character varying AS
$BODY$
DECLARE
BEGIN
   return "Hi " || item;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
```

Now our Development and Production servers are different. We need to execute the last script in Production, but instead of doing it we are going to check differences with our pgdbsync script.

The usage of [pgdbsync](https://code.google.com/p/pgdbsync/) command line script is the following one:

 -c \[schema\]
 -f \[from database\]
 -t \[to database\]
 -a \[action: diff | summary | run\]

```commandline
./pgdbsync -s web -f devel -t prod -a summary
HOST : production :: prod1
--------------------------------------------
function
 create :: WEB.hello(varchar)
tables
 create :: WEB.test
view
 create :: WEB.testview
 
[OK]  end process
```

Here we can see our production server is different from the development one.

```commandline
./pgdbsync -s wf -f devel -t prod -a diff
HOST : production :: prod1
--------------------------------------------
CREATE OR REPLACE FUNCTION web.hello(item character varying)
 RETURNS character varying
 LANGUAGE plpgsql
AS $function$
DECLARE
BEGIN
 return "Hi " || item;
END;
$function$
 
CREATE TABLE web.test(
 test_name character NOT NULL,
 test_id integer NOT NULL,
 test_date timestamp without time zone NOT NULL,
 CONSTRAINT pk_test PRIMARY KEY (test_date)
)
TABLESPACE web;
ALTER TABLE web.test OWNER TO user;
 
CREATE OR REPLACE VIEW web.testview AS
 SELECT test.test_name, test.test_id, test.test_date FROM web.test WHERE (test.test_name ~~ 't%'::text);;
ALTER TABLE web.testview OWNER TO user;
[OK]  end process
```

and finally

```commandline
./pgdbsync -s web -f devel -t prod -a run
HOST : production :: prod1
----------------------------------

[OK]  end process
```

And that's all. Our databases are synchronized. The library is not finished. Foreign keys are not supported yet, but it checks:

- Tables
- Constraints
- Sequences
- Views
- Functions

To finish the demo we are going to drop the new objects at development server and we will run cli script again:

```commandline
./pgdbsync -s wf -f devel -t prod1 -a diff
 
HOST : prododuction :: prod1
--------------------------------------------
 
drop function web.hello(varchar);
 
DROP TABLE web.test;
 
drop view web.testview;
 
[OK]  end process
```

Source code is available at Google, code [here](http://code.google.com/p/pgdbsync/).

An important post I've read to understand to PostgreSQL's information schema is [this one](http://alberton.info/postgresql_meta_info.html) from great Lorenzo Alberton. It really helped me building pgdbsync.

The Library is not finished yet and it may crashed in some cases (not all data-types are covered). I always check diff file before execute it. If you want to join me to develop the library, don't hesitate to contact me :).
