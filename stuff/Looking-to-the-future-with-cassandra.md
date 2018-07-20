# Looking to the future with Cassandra

Digg has been researching ways to scale our database infrastructure for some time now. We’ve adopted a traditional vertically partitioned master-slave configuration with MySQL, and also investigated sharding MySQL with IDDB. Ultimately, these solutions left us wanting. In the case of the traditional architecture, the lack of redundancy on the write masters is painful, and both approaches have significant management overhead to keep running.

Since it was already necessary to abandon data normalization and consistency to make these approaches work, we felt comfortable looking at more exotic, non-relational data stores. After considering HBase, Hypertable, Cassandra, Tokyo Cabinet/Tyrant, Voldemort, and Dynomite, we settled on Cassandra.

Each system has its own strengths and weaknesses, but Cassandra has a good blend of everything. It offers column-oriented data storage, so you have a bit more structure than plain key/value stores. It operates in a distributed, highly available, peer-to-peer cluster. While it’s currently lacking some core features, it gets us closer to where we want to be than the other solutions.

We started thinking seriously about deploying Cassandra in production around three weeks ago. After looking at the site for something that would be a good fit, we settled on green badges. These badges appear on the Digg icon for a story when one of your friends has dugg it. This has been a problematic feature to support as we’ve grown, and corners had to be cut to keep it from breaking. They’re disabled on large pages (such as top in 365) and highly digg stories.

The fundamental problem is endemic to the relational database mindset, which places the burden of computation on reads rather than writes. This is completely wrong for large-scale web applications, where response time is critical. It’s made much worse by the serial nature of most applications. Each component of the page blocks on reads from the data store, as well as the completion of the operations that come before it.

Non-relational data stores reverse this model completely, because they don’t have the complex read operations of SQL. The model forces you to shift your computation to the writes, while reducing most reads to simple operations – the equivalent of `SELECT * FROM ´Table´`.

### The Problem

In both models, we’re computing the intersection of two sets:

1. Users who dugg an item.
2. Users that have befriended the digger.

### The Relational Model

The schema for this information in MySQL is:

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

The Friends table contains many million rows, while Diggs holds hundreds of millions. Computing the intersection with a JOIN is much too slow in MySQL, so we have to do it in PHP. The steps are:

1. Query Friends for all my friends. With a cold cache, this takes around 1.5 seconds to complete.
2. Query Diggs for any diggs of a specific item by a user in the set of friend user IDs. This query is enormous, and looks something like:
```sql
SELECT `digdate`, `id` FROM `Diggs`
 WHERE `userid` IN (59, 9006, 15989, 16045, 29183,
                    30220, 62511, 75212, 79006)
   AND itemid = 13084479 ORDER BY `digdate` DESC, `id` DESC LIMIT 4;
```

The real query is actually much worse than this, since the IN clause contains every friend of the user, and this can balloon to hundreds of user IDs. A full query can actually clock in at 1.5kb, which is many times larger than the actual data we want. With a cold cache, this query can take *14 seconds* to execute.

Of course, both queries are cached, but due to the user-specific nature of this data, it doesn’t help much.

### The Non-Relational Model

Since non-relational data stores don’t have schemas in the same sense as relational databases, it’s harder to explain the Cassandra model. Non-relational schemas are much more flexible, and they’re mostly convention.

What we did is create a set of buckets per (user, item) pair, with each bucket containing a list of users who dugg the item who are also friends of the viewing user. With this layout, reads don’t require computation.

When an item is dugg, we asynchronously populate Cassandra. This job fetches the list of followers of the digging user, and places one column in each of their buckets. This is a large amount of data for popular users. Kevin Rose, for example, has 40,000 followers. Thanks to Cassandra’s excellent write performance and batch operations, every column is inserted at once, atomically, in under a second.

There’s no such thing as a free lunch, of course, and the fundamental trade-off in this model is CPU vs. disk. We have to store the computed results on disk, rather than generating them on the fly. It’s an acceptable trade-off in our case, since disks are cheap and scaling SQL is very hard.

For this feature, the fully denormalized Cassandra dataset weighs in at 3 terabytes and 76 billion columns.

### Conclusion

We haven’t made any secret of our interest in NOSQL in general and Cassandra specifically. We believe in this technology, and we are contributing to its ongoing development, both by submitting patches and by funding development of features necessary to support wide scale deployment.

This is the first thing we’ve migrated to Cassandra, but it definitely won’t be the last.

