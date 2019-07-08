---
layout: post
title: Bloom Filters for Table Lookups
date: 2019-07-08 13:55:00 +02:00
---

{{ page.title }}
================

**What is a bloom filter?** A [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) is a bit-array of a fixed size `m`. A (hash-) function is defined, which gives `k` different values within the range `[0...m-1]`. Two functions are defined on this structure, `add()` and `lookup()`

Pseudo-Code for a `m=1024, k=3`-Bloom Filter:
```go 
var filter [1024]bool

func hash(key string) (k0,k1,k2 int)

func add(key string) {
    k0,k1,k2 := hash(i)
    filter[k0] = true
    filter[k1] = true
    filter[k2] = true
}

func lookup(key string) bool {
    k0,k1,k2 := hash(i)
    return filter[k0] && filter[k1] && filter[k2]
}
```

A Figure of a `m=18, k=3`-Bloom Filter.

![Bloom Filter](https://upload.wikimedia.org/wikipedia/commons/thumb/a/ac/Bloom_filter.svg/640px-Bloom_filter.svg.png)

### Using a Bloom Filter for Table Lookups

Let's apply these Rules to a Table. In this (stupid) example, we have one key-value pair per row and one key per bloom filter. Our bloom filter will be `m=1024, k=3`!

```sql
-- SQL-Pseudocode.
CREATE TABLE bloom_table_v1 (
    row_bloom bit[1024], -- per-row bloom filter.
    row_key text,
    row_value text
);
```

Now, lets code a Lookup.

```sql
SET lookup_key = ?::text;
SET k0,k1,k2 = hash(lookup_key);
SELECT row_value FROM (
    SELECT * FROM bloom_table_v1
    WHERE row_bloom[k0] AND row_bloom[k1] AND row_bloom[k2]
) WHERE row_key = lookup_key;
```

Now lets code another table, where we have multiple key-value pairs per row.

```sql
-- SQL-Pseudocode.
CREATE TABLE bloom_table_v2 (
    row_bloom bit[1024], -- per-row bloom filter.
    row_kvs hstore
);
```

And the lookup code for this.

```sql
SET lookup_key = ?::text;
SET k0,k1,k2 = hash(lookup_key);
SELECT row_kvs[lookup_key] FROM bloom_table_v2
WHERE row_bloom[k0] AND row_bloom[k1] AND row_bloom[k2];
```

#### Using compressed Bitmaps

So, while the algorithm showed above is a stupid linear search, we can represent the 1024-wide bit-array as a collection of columns (`row_bloom[0],row_bloom[1],...,row_bloom[1022],row_bloom[1023]`), each represented as a compressed bitmap (such as [Roaring Bitmaps](http://roaringbitmap.org/)), on which boolean set-operations can be computed efficiently.

Using this technique, in some cases, we can achieve a considerably greater performance and compactness, than the use of a B-Tree. This technique can also handle sparse indeces with ease.

Note: The write-performance can be considerably lower, as we need to update alot of columns on every insert.