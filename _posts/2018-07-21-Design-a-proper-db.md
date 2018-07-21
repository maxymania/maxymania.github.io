---
layout: post
title: How to design a proper DB schema for dynamic web pages.
date: 2018-07-21 13:45:00 +02:00
---

# {{ page.title }}

### What is normalization

There is an excellent Article about [Database normalisation on Wikipedia](https://en.wikipedia.org/wiki/Database_normalization).
Normalization is to eliminate [Transitive dependencies](https://en.wikipedia.org/wiki/Transitive_dependency)
as well as insertion, update, and deletion anomalies.

Informally, a relational database relation is often described as "normalized" if it meets 3NF (third normal form).

### How RDBMS and normalization came to the web world

The role of RDBMSes in dynamic websites is closely related with the beginnings of PHP and MySQL in the 90s.
Before PHP gained traction, websites have been mostly written in Perl. PHP initially began as a set of CGI programs written in Perl
known as PHP/FI (Personal Home Page Tools/Forms Interpreter).
MySQL arose to be the first (propably) Free and Open Source SQL database (Postgres didn't support SQL at the time),
furthermore MySQL was built with to be compatible to the Semi-Free mSQL dbms, which was free for personal use, and therefore popular.

Within the evolution of the WWW, mSQL and later MySQL was extensively used in dynamic web pages, which have been written in Perl
and later more and more PHP. Initial systems used MySQL merly as a document database with SQL interface, but as these web applications
(or CMS or Wiki engines) became more and more complex, heavily normalized database models had been implemented, as this is the
most obvious thing to do with an RDBMS.

### How to really "normalize" your data web applications

**A denormalized Table:**

**Table** `Blog_Posts`
id | title | body | category | tag
--- | - | - | - | -
1 | A paradox on ... | *...some text...* | Poetry | Paradox
1 | A paradox on ... | *...some text...* | Philosophy | Paradox
1 | A paradox on ... | *...some text...* | Poetry | Ideas
1 | A paradox on ... | *...some text...* | Philosophy | Ideas

This table is obviously denormalized (or unnormalized).
Everyone will propably agree, that this is a waste of diskspace.

**The "normalized" solution:**

**Table** `Blog_Posts`
id | title | body
--- | - | - 
1 | A paradox on ... | *...some text...*

**Table** `Blog_Post_Categories`
id | category
--- | -
1 | Poetry
1 | Philosophy

**Table** `Blog_Post_Tags`
id | tag
--- | -
1 | Paradox
1 | Ideas

This is the obvious solution as ruled by the 3NF (I think).

**My solution:**

**Table** `Blog_Posts`
id | title | body | categories | tags
--- | - | - | - | -
1 | A paradox on ... | *...some text...* | {Poetry,Philosophy} | {Paradox,Ideas}

This example violates even the 1NF (first normal form) by having multiple values per field.
However modern RDBMS usually have [array types](https://www.postgresql.org/docs/9.5/static/arrays.html).
And having arrays within fields is not expensive (at least not in PostgreSQL).

Despite not adhering to the normal forms, this example does'nt suffer from the kind of data explodification
as seen in the "denormalized" example above. Yet, it does'nt suffer from type of segmentation as the "normalized" example.

My solution also improves the data locality and CPU overhead. And, ironically, this solution saves more disk space than seperate tables.

Disclaimer: This works since a blog post is usually associated to less than 10 categorys and tags, rather than tousands of those.

### Never ever Join your Tables...

...like this, but this is obvious!

The query:

```sql
SELECT p.id AS id,title,body,category,tag
FROM Blog_Posts p
JOIN Blog_Post_Categories c ON c.d = p.id
JOIN Blog_Post_Tags t ON t.id = p.id;
```

is going to give you:

id | title | body | category | tag
--- | - | - | - | -
1 | A paradox on ... | *...some text...* | Poetry | Paradox
1 | A paradox on ... | *...some text...* | Philosophy | Paradox
1 | A paradox on ... | *...some text...* | Poetry | Ideas
1 | A paradox on ... | *...some text...* | Philosophy | Ideas

All your application wants is the `Blog_Posts` record + the list of
categories, the Blog-post is associated with + a list of of tags, the Blog-post
is associated with.

What your application doesn't want is this giant table-block. Those table-blocks
are interesting for automated report generation in the finance industry, but they
are not interesting for web developers.

### And again, be careful with your queries...

Revisiting the [Digg-Scenario](/stuff/Looking-to-the-future-with-cassandra.html).

Lets suppose we have the following schema...
```sql
CREATE TABLE Diggs (
  id      integer primary key,
  itemid  integer,
  userid  integer,
  digdate timestamp
);
CREATE TABLE Friends (
  id           serial primary key,
  userid       integer,
  username     text,
  friendid     integer,
  friendname   text,
  mutual       boolean,
  date_created timestamp,
  -- Constraints...
  unique (userid,friendid)
);
```
...and the following indexes:
```sql
CREATE INDEX ON Diggs (itemid,userid);
CREATE INDEX ON Friends (userid);
```

For a given user-id, we want to get the friends of the friends, and from those, we want
any Diggs of an Item.

```sql
SELECT d.digdate,d.id
FROM Friends mf,Friends fomf,Diggs d
WHERE
	mf.userid = $myself AND
	fomf.userid = mf.friendid AND
	d.userid = fomf.friendid AND
	d.itemid = $specific_item
ORDER BY d.digdate DESC, d.id DESC
LIMIT 4;
```

To the average user, it should be obvious how the query plan should look like:
go straight through the indexes! But take a look at the query plan: If you
request the query plan using [explain](https://www.postgresql.org/docs/9.6/static/using-explain.html)
or an equivalent command, you will often find something bizarre.

In large scale web applications, joins are often pulled into the application.


