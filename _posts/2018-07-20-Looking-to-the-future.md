---
layout: post
title: On 'Looking to the future with Cassandra'
date: 2018-07-20 16:07:00 +02:00
---

{{ page.title }}
================

Revisiting the legendary switch of Digg to Cassandra.
There was a Blog post made by Digg [Looking to the future with Cassandra](/stuff/Looking-to-the-future-with-cassandra.html)
as well as a response from the RDBMS camp [The Impact of SSDs on Database Performance and the Performance Paradox of Data Explodification](https://dennisforbes.ca/index.php/2010/03/24/the-impact-of-ssds-on-database-performance-and-the-performance-paradox-of-data-explodification/).

### The MySQL problem

```sql
CREATE TABLE `Diggs` (
  `id`      INT(11),
  `itemid`  INT(11),
  `userid`  INT(11),
  `digdate` DATETIME,
  PRIMARY KEY (`id`),
  KEY `user`  (`userid`),
  KEY `item`  (`itemid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
 
CREATE TABLE `Friends` (
  `id`           INT(10) AUTO_INCREMENT,
  `userid`       INT(10),
  `username`     VARCHAR(15),
  `friendid`     INT(10),
  `friendname`   VARCHAR(15),
  `mutual`       TINYINT(1),
  `date_created` DATETIME,
  PRIMARY KEY                (`id`),
  UNIQUE KEY `Friend_unique` (`userid`,`friendid`),
  KEY        `Friend_friend` (`friendid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

The Friends table contains many million rows, while Diggs holds hundreds of millions. Computing the intersection with a JOIN is much too slow in MySQL, so they have to do it in PHP. The steps are:

**Step 1:** Query Friends for all my friends. It will *propably* look like this:

```sql
SELECT `friendid` FROM `Friends`
	WHERE `userid` IN (353, 9006, 15989, 75212);
```
or like this:
```sql
SELECT `userid` FROM `Friends`
	WHERE `friendid` IN (353, 9006, 15989, 75212);
```

With a cold cache, this is said to take around **1.5 seconds** to complete.

**Step 2:** Query Diggs for any diggs of a specific item by a user in the set of friend user IDs. This query looks something like this:
```sql
SELECT `digdate`, `id` FROM `Diggs`
	WHERE `userid` IN (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
	AND `itemid` = 13084479 ORDER BY `digdate` DESC, `id` DESC LIMIT 4;
```

With a cold cache, this is said to take around **14 seconds** to complete.

### Why is it so slow?

In the Table `´Diggs´` both `´userid´` and `´itemid´` are indexed (called "KEY" in MySQL). However, they are indexed individually.

For simplicity, I deliberately ignore the clauses `ORDER BY ´digdate´ DESC, ´id´ DESC LIMIT 4`!

We have two clauses
```sql
WHERE `userid` IN (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
```
and
```sql
WHERE `itemid` = 13084479
```

which both can be implemented as index scans, so, logically, the "ideal" query plan would look like this:
```sql
SELECT `digdate`, `id`
  FROM `Diggs`
  WHERE `id` IN (
     -- An index-only scan on KEY `user`  (`userid`)
     SELECT `id` FROM `Diggs` WHERE `userid` IN
       (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
   INTERSECT
     -- An index-only scan on KEY `item` (`itemid`)
     SELECT `id` FROM `Diggs` WHERE `itemid` = 13084479
  )
```

To execute this obvious query plan, MySQL had to create the intersection of two unsorted streams of integers, that are returned by the index-scans.
I think this is sloq and (almost) impossible to execute for MySQL.

So instead the query plan will in fact look like this:
```sql
SELECT `digdate`, `id`
  FROM (
   -- An index scan on KEY `user` (`userid`)
   SELECT * FROM `Diggs` WHERE `userid` IN
    (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
  ) AS `Diggs`
WHERE `itemid` = 13084479 -- Perform a linear search on the result set of the index scan.
;
```
or this:
```sql
SELECT `digdate`, `id`
  -- An index scan on KEY `item` (`itemid`)
  FROM (SELECT * FROM `Diggs` WHERE `itemid` = 13084479) AS `Diggs`
  -- Perform a linear search on the result set of the index scan.
WHERE `userid` IN (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
;
```

These query plans involve a linear search on a huge amount of data, despite involving Indexes.

### The solution

The solution on this problem is to create a new index as follows:
```sql
CREATE INDEX `Diggs_new_index` ON `Diggs` (`userid`,`itemid`);
```

The resulting query plan will look like this:

```sql
SELECT `digdate`, `id`
	FROM `Diggs`
	WHERE `id` IN (
		SELECT `id` FROM `Diggs` WHERE (`userid`,`itemid`) IN (
			(59,13084479),
			(9006,13084479),
			(15989,13084479),
			(16045,13084479),
			(29183,13084479),
			(30220,13084479),
			(62511,13084479),
			(75212,13084479),
			(79006,13084479)
		) -- An index-only scan on INDEX `Diggs_new_index` (`userid`,`itemid`)
	)
```

Of course, it depends on the query planer create a good query plan. I remember times,
when i was sitting in front of a database trying out different implementations of a
specific query until the query plan looked like, what i envisioned it to look like.

### What is Cassandra for?

At the time when Digg switched to Cassandra, it was a wide-column store with a data-model similar to BigTable.
Until today, it has evolved into something that has a Query-Language, that has a data-model similar to an RDBMS.

I would even argue, that Cassandra ultimately is an RDBMS (which differs from traditional transactional SQL databases, of course).
The big strength of Cassandra is, that it specifically designed to be distributed, being developed after
the [Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), as published by Amazon,
which's [primary problem was Availability](https://www.allthingsdistributed.com/2017/10/a-decade-of-dynamo.html)
and not performance, like in Diggs case.

As of today a Cassandra cluster can serve a even a heavily normalized dataset.
You can even perform joins, if you use a middleware like the *MySQL to NoSQL gateway* of [yoursql](https://github.com/a-mail-group/yoursql),
my pet project, or any other solutin for this, that is out there (such as the "federated" storage engine of MySQL/MariaDB).


