---
layout: post
title: Boosting Digg with Cassandra
date: 2018-07-25 11:52:00 +02:00
---

# {{ page.title }}

Another look into the problem described in
[Looking to the future with Cassandra](/stuff/Looking-to-the-future-with-cassandra.html)
reveals several optization opportunities.

They had a MySQL-Table defined as:

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
```

... and containing hundreds of millions of rows. And they were performing a query, that looks like this:

```sql
SELECT `digdate`, `id` FROM `Diggs`
 WHERE `userid` IN (59, 9006, 15989, 16045, 29183,
                 30220, 62511, 75212, 79006)
AND itemid = 13084479 ORDER BY `digdate` DESC, `id` DESC LIMIT 4;
```

... which was said to take *14 seconds* on a cold cache.

Hi had written a [detailed analysis of their problem](https://maxymania.github.io/2018/07/20/Looking-to-the-future.html).
But i still think that there is a certain optimization opportunity for this scenario.

### Using Cassandra as DB for Diggs

These days, Cassandra has a Query-Language akin to SQL. That means, you can define Tables just like
in any RDBMS (with a few limitations, of course). Aside the limitations, there are also some "soft rules"
that SHOULD be followed ([Basic Rules of Cassandra Data Modeling](https://www.datastax.com/dev/blog/basic-rules-of-cassandra-data-modeling)):

- **Rule 1: Spread Data Evenly Around the Cluster**
- **Rule 2: Minimize the Number of Partitions Read**

**You should really adhere to Rule 1.** What you really don't want to have is a skew data distribution.
But you can look at Rule 2 more relaxed. Think of a Bulk-Lookup of many Items from a Key-Value-Store.
This would pretty much violate Rule 2.

**First attempt:**

```sql
CREATE TABLE Diggs (
  "id"    int,
  itemid  int,
  userid  int,
  digdate timestamp,
  PRIMARY KEY((itemid,userid),digdate)
) WITH CLUSTERING ORDER BY (digdate DESC);
```

(I assumed, that a user can digg the same item multiple times.)

```sql
SELECT digdate, "id" FROM Diggs
WHERE itemid = 13084479
AND userid IN (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
ORDER BY digdate DESC LIMIT 4 ALLOW FILTERING;
```

The problem here is, that *Rule 2* is violated (probably). The query defines many partition keys,
probably hundreds. This could put a strain onto the entire cluster.
(Aside: I think, that many people of the RDBMS camp would discourage, large cross-shard querys as well.)

**Second attempt:**

```sql
CREATE TABLE Diggs (
  "id"    int,
  itemid  int,
  userid  int,
  digdate timestamp,
  PRIMARY KEY(itemid,userid)
);
```

(If a user diggs an item, (s)he alreadi digged, this will simply overwrite the existing row.)

```sql
SELECT digdate, "id" FROM Diggs
WHERE itemid = 13084479
AND userid IN (59, 9006, 15989, 16045, 29183, 30220, 62511, 75212, 79006)
ORDER BY digdate DESC LIMIT 4 ALLOW FILTERING;
```

The problem here is, that *Rule 1* might be violated, however, since there are really a lot of diggable items
(probably), that issue might be amortized pretty much. If that assumption is true, the data distribution
within the cluster isn't necessarily more skewed, than in the first attempt.
Yet, this example only uses one hash partition (as opposed to potentially hundreds).
That might be an order of magnitude faster.

I think, this is a truly elegant solution.

### PostgreSQL versus Cassandra

In PostgreSQL you are much less constrained on how you can query your database.
As a full-featured RDBMS, PostgreSQL is also capable of doing joins including
query optimisation. On the other Hand, Cassandra offers an Amazon Dynamo-Style
clustering feature, and is partition-tolerant and always writable. This is desireable
on workloads like this, since this improves uptime and distributes the load
onto many nodes.

### Conclusion

Cassandra lacks some features, found in regular (relational) database products such as PostgreSQL,
but in can offer you a nicely distributet, elastic, fault-tolerant database.

