---
datePublished: '2016-11-14T09:39:03.486Z'
sourcePath: _posts/2016-11-13-sometimes-hbase-is-not-the-answer.md
inFeed: true
hasPage: true
author: []
via: {}
dateModified: '2016-11-14T09:39:03.008Z'
title: Sometimes HBase is not the answer
publisher: {}
description: ''
starred: false
url: sometimes-hbase-is-not-the-answer/index.html
_type: Article

---
# Sometimes HBase is not the answer
![](https://the-grid-user-content.s3-us-west-2.amazonaws.com/45a7a6be-128c-4bef-b582-fcd21bd8c8e0.png)

---

HBase seems to be purported by the be all and end all of Hadoop based data storage, but due to differing access patterns this not always the case.

HBase is a distributed non-relational database based on Google's BigTable. It stores data as a column-oriented key-value data store. Mainly used as a input and output to Map Reduce jobs it seemed as evolution to use it with the next generation of Execution engines, sometimes this can work efficiently sometimes it does not.

In my role I was working with a previous implementation of data stored in Hbase, and was recommended that I use this with a newly developed Spark application. This application performed analysis over a set time period meaning that the access needed to provided based on time.
![](https://s3-us-west-2.amazonaws.com/the-grid-img/p/82432499344a73c5477268f3dca8afcac816aafd.gif)

Checking out the data I realised that we might get into a bit of trouble.

## Now the trouble starts?

The key was made up of the following:

<Product ID\>, <Version Timestamp as a string representation\>,<Order Number\>

So I have to retrieve the keys that have a timestamp within the required time range... In string format nonetheless.

A highlight here is to let the reader know that with HBase you get one very efficient index. This index is the row key, and if you cannot reconstruct the key then you may be party to having to perform a full table scan.

First question here is why would you not just use a unix based time stamp as a prefix so you can easily define the range as keys lexicographically ordered.

When you write all the new rows sequentially, they all end up being on the same region server, because they are sorted and this forces them to be close to each other. It is true that there is sharding, and moreover, there is automatic sharding. This means that the new regions (the areas of the hard drive where the data is written) will eventually come into play, but this happens only later. Right now, you have got a hotspot. Practically, you won't notice this under low write speeds. If you have less than a hundred writes per second, this is not important since your RegionServer copes quite all right. This number (hundred writes per second) might change on your hardware and HBase, but it will be in the low range, because it means that you are not utilizing your entire HBase cluster but only one region server. The Product ID prefix is to load balance the writes and reads to reduce hotspotting.

Even though I was wary about the full table scan, and could not find another way to perform the scan to be more performant.

Then it happened on our testing cluster, when a table was too big the Spark job would receive a RPC timeout (60s). So now we NEEDED to find another way.

I was then informed that we not only store our data in Hbase we also use a lily indexer solution to index the keys and other key data in Solr.

In the month ahead I spent a lot of my non-project based time finding out how to retrieve large amounts of data from Solr in a timely fashion. Do not get me wrong Solr is great when you are performing intensive text searches on indexed data, but Solr falls flat on its face and will CRASH if you try and retrieve a lot of data. In the time of performing this investigation I looked into Solr Deep pagination, using stream and response, none really worked if the timeliness of the data access will turn from seconds to many minutes.

## HBase is not that into you

So coming to the realisation that our access pattern where we were reliant exclusively on time series data and we collected this data wholesale, not using filtering by Product ID or any other metric meant that HBase just was not the pancea for all Hadoop based data requirements.

We in turn then looked further afield at newer technologies that allowed for the access patterns we had in mind.

### Kudu

Plugs the gap between HDFS and HBase helping users with hybrid architectures have the best performance. So it provides a quick, performant single storage layer for our engines to use.

### Apache Phoenix

This can be done in Phoenix (very tightly integrated with Hbase), even though it is originally meant for OLTP this can be useful to provide an immutable secondary index for Hbase. It also has a connector for Spark (this has been rejected, I believe for the reasoning that this is another new technology).

### Hive and Imapala

Hive and Impala can be used to read the data within HDFS (not Hbase as I have been told that the M+R jobs required to retrieve the data from the tables could put Hbase at risk).

Although some people think only of the SQL interface, this makes sense, as although it is slower than Hbase for random access retrieval, it is great for Scans.

Impala pushes out ahead of Hive as it avoids latency, by circumventing MapReduce to directly access the data through a specialised distributed query engine that is very similar to those found in commercial parallel RDBMSs.

The result is order-of-magnitude faster performance than Hive, depending on the type of query and configuration.

### Parquet

This is a compressed, columnar format file, with the added benefit of a schema to allow for direct querying. It also is quite efficient in terms of disk I/O and memory utilisation. It also has the added benefit that, if combined with Impala (Hive is not as compatible with parquet), the query engine is further optimised meaning for speedier results.

# The solution

We went with a process of utilising parquet files to build tables in Impala.

As of next year Cloudera promises support for Kudu, so I am excited to get to have a go once it is officially released.