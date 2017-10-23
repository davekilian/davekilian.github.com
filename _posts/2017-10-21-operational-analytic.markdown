---
layout: post
title: Operational vs Analytic Databases
author: Dave
draft: true
---

Brain dump:

Originally was trying to define OLTP vs OLAP, but I decided I don't like those terms much so change them.

OLTP vs OLAP

Best explanation I've found so far is here: http://datawarehouse4u.info/OLTP-vs-OLAP.html

Even then it's kind of hard to follow though. If I understand correctly, it should be a bit easier to explain this:

OLTP and OLAP refer to different types of workloads databases can be tuned for.
OLTP workloads push a huge number of simple transactions
OLAP workloads push a smaller number of very complex transactions
Any database engine can be used for either type of workloads, but usually database engines are tuned for one type of workload or the other. An "OLTP database" is tuned for OLTP-style workloads possibly at the expensive of OLAP, for example.

Generally, OLTP workloads appear as part of day-to-day operations, whereas OLAP workloads appear when you try to analyze your data to find insights.

Say, for example, you're a credit card company. For a number of reasons you need to track all the transactions customers have charged on all of their credit cards. This system needs to be performant even under spiky loads (e.g. customers will be using their credit cards more often at lunchtime than at 3 AM). However, the queries aren't all that complicated: for each transaction we just need to insert a row specifying the customer whose credit card was charged, the recipient of the money, and the number of dollars spent. The simplicity of the queries coupled with the sheer number of queries being executed makes this an OLTP workload.

Say we also want to build a recommendation engine that advertises to customers new businesses they may be interested in based on their purchasing habits. To implement this, we spin up a second database system and offload the transaction information from the previous paragraph to our new database engine asynchronously (so as not to affect the performance of the primary transactions engine). Once this data has been offloaded, we can run complex algorithms to discover patterns in purchasing habits in order to generate recommendations for each cardholder. These queries are quite expensive, but we only need to run them once in a while. The complexity of the queries coupled with the low number of queries being executed makes this an OLAP workload.

In practice, OLTP and OLAP workloads follow the pattern above: a business or application which collects large amounts of data uses an OLTP database to collect and store the data, and asynchronously copies the data into an OLAP database for 'big data' analytics. 

---

Additionally

OLTP workloads tend to focus on rows (pit this row in the database as fast as you can, look up this row as fast as you can)

OLAP workloads tend to focus on aggregating columns and generating new relationships between rows. 

---

OLTP: high throughput of simple queries

OLAP: efficient execution of complex queries 

---

To understand the term, need to understand a bit of history: around the same time 'big data' became a thing and so did the cloud, and part of big data meant analyzing large troves of information to produce insights. This gave rise to the need for new types of databases: previously most existing databases were more OLTP-like, but many of the newer NoSQL kinds were optimized for OLAP.

As a result, whenever you see the distinction between the two types of databases, you'll always see 'analytical' for the newer type of database, but terminology varies for the 'older' time (transactional, operational, etc)
