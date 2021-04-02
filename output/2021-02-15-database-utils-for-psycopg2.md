---
title: "Database utils for psycopg2"
date: "2021-02-15"
categories: 
  - "technology"
tags: 
  - "postgresql"
  - "python"
---

I normally need to perform raw queries when I'm working with Python. Even when I'm using Django and it's ORM I need to execute sql against the database. As I use (almost always) PostgreSQL and I use (as we all do) psycopg2. Psycopg2 is very complete and easy to use but sometimes I need a few helpers to make my usual tasks easier. I don't want to create a complex library on top of psycopg2, only a helper.

Something that I miss from PHP and DBAL library is the way that DBAL helps me to create simple CRUD operations. The idea of this helper functions is to do something similar. Let's start

I've created procedural functions to do all and also a Db class. Normally all functions needs the cursor parameter.

def fetch\_all(cursor, sql, params=None):
    cursor.execute(
        query=sql,
        vars={} if params is None else params)

    return cursor.fetchall()

The Db class accepts cursor in the constructor

db = Db(cursor=cursor)
data = db.fetch\_all("SELECT \* FROM table")

## Connection

Nothing especial in the connection. I like to use "connection\_factory=[NamedTupleConnection](https://www.psycopg.org/docs/extras.html?highlight=namedtupleconnection#psycopg2.extras.NamedTupleConnection)" to allow me to access to the recordset as an Object. I've created simple factory helper to create connections:

def get\_conn(dsn, named\_tuple=False, autocommit=False):
    conn = psycopg2.connect(
        dsn=dsn,
        connection\_factory=NamedTupleConnection if named\_tuple else None,
    )
    conn.autocommit = autocommit

    return conn

Sometimes I use [NamedTuple](https://www.psycopg.org/docs/extras.html?highlight=namedtuple#namedtuple-cursor) in the cursor instead connection, so I've created a simple helper

def get\_cursor(conn, named\_tuple=True):
    return conn.cursor(cursor\_factory=NamedTupleCursor if named\_tuple else None)

## Fetch all

db = Db(cursor)
for reg in db.fetch\_all(sql="SELECT email, name from users"):
    assert 'user1' == reg.name
    assert 'user1@email.com' == reg.email

## Fetch one

data == db.fetch\_one(sql=SQL)

## Select

data = db.select(
        table='users',
        where={'email': 'user1@email.com'}

This helper only allows me to perform simple where statements (joined with AND). If I need one complex Where, then I use fetch

## Insert

Insert one row

db.insert(
        table='users',
        values={'email': 'user1@email.com', 'name': 'user1'})

Or insert multiple rows (using spycopg2's [executemany](https://www.psycopg.org/docs/cursor.html#cursor.executemany))

db.insert(
        table='users',
        values=\[
            {'email': 'user2@email.com', 'name': 'user2'},
            {'email': 'user3@email.com', 'name': 'user3'}
        \])

We also can use insert\_batch to insert all rows within one command

db.insert\_batch(
        table='users',
        values=\[
            {'email': 'user2@email.com', 'name': 'user2'},
            {'email': 'user3@email.com', 'name': 'user3'}
        \])

## Update

db.update(
        table='users',
        data={'name': 'xxxx'},
        identifier={'email': 'user1@email.com'},
    )

## Delete

db.delete(table='users', where={'email': 'user1@email.com'})

## Upsert

Sometimes we need to insert one row or update the row if the primary key already exists. We can do that with two statements: One select and if there isn't any result one insert. Else on update. We can do that with one sql statement. I've created a helper to to that:

def \_get\_upsert\_sql(data, identifier, table):
    raw\_sql = """
        WITH
            upsert AS (
                UPDATE {tbl}
                SET {t\_set}
                WHERE {t\_where}
                RETURNING {tbl}.\*),
            inserted AS (
                INSERT INTO {tbl} ({t\_fields})
                SELECT {t\_select\_fields}
                WHERE NOT EXISTS (SELECT 1 FROM upsert)
                RETURNING \*)
        SELECT \* FROM upsert
        UNION ALL
        SELECT \* FROM inserted
    """
    merger\_data = {\*\*data, \*\*identifier}
    sql = psycopg2\_sql.SQL(raw\_sql).format(
        tbl=psycopg2\_sql.Identifier(table),
        t\_set=psycopg2\_sql.SQL(', ').join(\_get\_conditions(data)),
        t\_where=psycopg2\_sql.SQL(' and ').join(\_get\_conditions(identifier)),
        t\_fields=psycopg2\_sql.SQL(', ').join(map(psycopg2\_sql.Identifier, merger\_data.keys())),
        t\_select\_fields=psycopg2\_sql.SQL(', ').join(map(psycopg2\_sql.Placeholder, merger\_data.keys())))

    return merger\_data, sql

def upsert(cursor, table, data, identifier):
    merger\_data, sql = \_get\_upsert\_sql(data, identifier, table)

    cursor.execute(sql, merger\_data)
    return cursor.rowcount

Now we can execute a upsert in a simple way:

db.upsert(
        table='users',
        data={'name': 'yyyy'},
        identifier={'email': 'user1@email.com'})

## Stored procedures

data = db.sp\_fetch\_one(
    function='hello',
    params={'name': 'Gonzalo'})

data = db.sp\_fetch\_all(
    function='hello',
    params={'name': 'Gonzalo'})

## Transactions

In PHP and [DBAL](https://www.doctrine-project.org/projects/doctrine-dbal/en/2.12/reference/transactions.html) to create one transaction we can use this syntax

<?php
$conn->transactional(function($conn) {
    // do stuff
});

In Python we can do the same with context management, but I prefer to use this syntax:

with transactional(conn) as db:
    assert 1 == db.insert(
        table='users',
        values={'email': 'user1@email.com', 'name': 'user1'})

The transactional function is like that (I've created two functions. One with raw cursor and another one with my Db class):

def \_get\_transactional(conn, named\_tuple, callback):
    try:
        with conn as connection:
            with get\_cursor(conn=connection, named\_tuple=named\_tuple) as cursor:
                yield callback(cursor)
            conn.commit()
    except Exception as e:
        conn.rollback()
        raise e

@contextmanager
def transactional(conn, named\_tuple=False):
    return \_get\_transactional(conn, named\_tuple, lambda cursor: Db(cursor))

@contextmanager
def transactional\_cursor(conn, named\_tuple=False):
    return \_get\_transactional(conn, named\_tuple, lambda cursor: cursor)

## Return values

Select and fetch helpers returns the recordset, but insert, update, delete and upsert returns the number of affected rows. For example that's the insert function:

def \_get\_insert\_sql(table, values):
    keys = values\[0\].keys() if type(values) is list else values.keys()

    raw\_sql = "insert into {tbl} ({t\_fields}) values ({t\_values})"
    return psycopg2\_sql.SQL(raw\_sql).format(
        tbl=psycopg2\_sql.Identifier(table),
        t\_fields=psycopg2\_sql.SQL(', ').join(map(psycopg2\_sql.Identifier, keys)),
        t\_values=psycopg2\_sql.SQL(', ').join(map(psycopg2\_sql.Placeholder, keys)))

def insert(cursor, table, values):
    sql = \_get\_insert\_sql(table=table, values=values)
    cursor.executemany(query=sql, vars=values) if type(values) is list else cursor.execute(sql, values)

    return cursor.rowcount

## Unit tests

You can try the library running the unit tests. I've also provide one docker-compose.yml file to set up a PostgreSQL database to run the tests.

version: '3.6'

services:
  pg:
    build:
      context: .docker/pg
      dockerfile: Dockerfile
    ports:
      - 5432:5432
    environment:
      POSTGRES\_PASSWORD: ${POSTGRES\_PASSWORD}
      POSTGRES\_USER: ${POSTGRES\_USER}
      POSTGRES\_DB: ${POSTGRES\_DB}
      PGDATA: /var/lib/postgresql/data/pgdata

## Install

You can install the library using pip

```
pip install dbutils-gonzalo123
```

Full code in my [github](https://github.com/gonzalo123/dbutils)
