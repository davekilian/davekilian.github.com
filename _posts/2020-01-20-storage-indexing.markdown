---
layout: post
title: Storage (for Mortals)
author: Dave
draft: true
---

*Never roll your own database!*

You've probably heard this advice before, or even took it for granted as common sense. But have you ever stopped to consider *why* you shouldn't build your own data storage engines?

After all, it can't be that *nobody* should build new file systems, databases, distributed search indexes and so on &mdash; times change, people change, hardware changes, and we aren't still using the original systems we designed back in the 1960s-70s. Clearly, some people do build their own data storage systems, and are pretty successful for doing so.

I'd argue the reason you generally shouldn't build one of these systems yourself is time: as a rule of thumb, it takes about 10 years to fully stabilize a new general-purpose data storage system to the point of being near-optimal and also bug-free, and for most projects you don't have that kind of time.

Some argue another reason not to build these systems ourselves is they're too complicated for us mortals to design, and that such things are best left to the *Experts* (TM). In this article, I want to show you this isn't true &mdash; it's not too hard to come up with the basic design of these systems as long as you have a good grasp of introductory data structures and algorithms.

My goal with this article is to walk you down a fairly simple line of reasoning that gets you more or less to today's state of the art in data storage. All you'll need is some basic familiarity with data structures like hash tables and binary trees.

So, let's design some storage systems!

## The Key Problem

What do you think is the main challenge of designing a data storage system?

Hint: ironically, the hard part of storage is not storing data. From a software perspective, storing data is pretty easy: you tell the hardware to store data and then you wait until the hardware is done. The hardware does all the work; your code just sets up the transfer and gets out of the way as fast as it can.

The hard part is retrieval. You have a disk full of data, and a user comes along and asks you for a specific record somewhere on that disk. How do you know where to go looking for it?

In other words, storage is a search problem!

And it's not just any search problem &mdash; it's key/value search, a problem you're already tackled when you learned basic data structures and algorithms.

Don't believe me? Every storage system uses a system of keys to identify and retrieve data, but a lot of systems obscure the key and make the key/value mapping less than obvious. But if you look close enough at any system that retrieves data, you'll find a key hiding somewhere in the system's design. For example:

* Want to download a web page? You need to provide its URL (a key)
* Want to read a file? You need to provide its path (a key)
* Want to load a value from memory? You need to provide its address (a key)
* Want to query a database? You need to provide a primary *key*

So at the end of the day, the main thing you're doing when you build a data storage system like a database, file system, cloud storage, etc is designing a map data structure which is stored on disk, alongside the actual keys and values the user asked you to store. Your system's write (store data) algorithm fills this map with key/value pairs and your system's read (retrieve data) algorithm uses this map to find the requested data. The goal is to design a set of algorithms where writes efficiently set up reads to also be efficient.

## Map Data Strutures

So now we know a storage system is an on-disk key-value map data structure and code for manipulating it. What does that map structure usually look like?

Well, let's start with some map data structures we already know:

### Hash Tables



### Binary Search Trees



## Maps for Disks

> Disks provide the same basic interface to the computer as memory does: an array of bytes:
>
> Diagram
>
> So the data structures we discussed above, which are typically built in memory, you can also store on a disk. 
>
> There's a catch though: disks might have the same interface as memory, but they perform quite differently!

### Disk Performance

> The different parts of your computer don't run at the same speed. 
>
> Copy table from, and link to https://www.prowesscorp.com/computer-latency-at-a-human-scale/

### Caching

> We see that RAM is a lot faster than disk, so maybe keep data in RAM instead of disk so we can retrieve it quickly. But there's a lot less space in RAM than there is on disk, so we have to cache selectively, guessing what will be useful (what will be read soon).
>
> Cache heuristics: "temporal locality:" workload likely to revisit data it used before. "spatial locality:" workload likely to iterate over keys in ascending or descending order.

### Hash Tables and Spatial Locality

> Since we want to cache data, hash tables aren't a good fit for our storage system. They purposefully scramble data in an attempt to minimize hash collisions:
>
> Diagram: `A B C D E -> 3 1 5 4 2 -> C A E D B`
>
> Binary trees keep data sorted in key order, maintaining a degree of spatial locality, making them closer to want we want out of a key-value mapping structure

### Another Observation

> For small I/O sizes, the time it takes to read from a disk is roughly constant
>
> Find a way to show the graphs we made in excel here
>
> Conclusion: a search data structure for disks should **minimize the number of disk operations**, even if that means each individual read operation reads more data than is strictly needed. This is true as long as the keys and values are small (<64KB maybe, per the graph above) 

### Binary Trees and Disk Accesses

> Binary trees have good asymptotic complexity, but the absolute number of disk accesses is very high:
>
> (20ms per read) x (25 reads per lookup) x (15 lookups) = 7.5 seconds (!)
>
> Imagine that was what it took to read just one of the posts in your Facebook/Twitter/Instagram/etc feed ... are you willing to wait minutes for your feed to load?

## B-Trees

> Why not take a binary search tree, and 'smoosh' it down by packing in many key/value pairs into a single node?
>
> Diagram
>
> We choose a node size around that sweet spot in the graph above (typically somewhere between 8K and 64K) and pack as many items into each node as will fit. We still use a binary search, but there are fewer individual nodes to read in, because each node has many values.
>
> To search, we read in the root node, binary-search it (as an array), and follow the child link to another node, which we then binary-search (as an array), recurisvely, until we either find the key being queried or an empty space where the key would be if we had it:
>
> Diagram
>
> Constructing a b-tree is a little more involved, but certainly nothing we can't handle Wikipedia has a good writeup (link)

### Nice Properties of B-Trees

> By packing many key-value records into a single node, we greatly reduce the number of individual nodes needed to store the tree. This allows us to store large amounts of data with a reasonable number of disk accesses.
>
> B-tree nodes are also cache friendly. A simple scheme is, every time you read in a node, you add it to an in-memory cache. This is simple, and gives you both spatial locality (because keys in a node are a sorted and adjacent) and temporal locality (becuase you're caching nodes that were read recently).

### The Ubiquitous B-Tree

> B-trees are a really good data structure for storage systems, and they're used for basically everything: databases, file systems, caches and more. 
>
> Many variants designed with different user cases in mind. Link to some
>
> Let's see how the interfaces and performance characteristics of many classes of storage systems fall out of b-trees:

## Let's Build a Database

> Overview showing what a database table looks like, with a diagram
>
> We can decompose a row into a key-value mapping by examininng the table schema
>
> Diagram
>
> A table, then, is a sorted array of this key-value mapping.
>
> So then the obvious approach is to store the database as a b-tree of rows.
>
> Diagram showing the example table on one side, the same rows in a b-tree on the other side
>
> Each record is a row, which can be decomposed into both a key and a value as described above.
>
> Let's look at how common queries work
>
> * Insert - show how this is a row insert
> * Select - with a single key
> * Select - with a range of keys
> * Delete - 
>
> To complete your database now all you need is a transaction manager (which locks ranges of keys that are being operated on, to provide isolation) and a query parser (giving users an interface to your database)
>
> Huzzah! We solved storage systems ... or did we? How do you handle data that's larger than what can fit in a single b-tree node?
>
> This is the purview of a different, but closely related kind of storage system . . .

## File Systems

> Databases vs file systems: similar in principle, differ in target workloads. Databases optimize for large numbers of relatively small 'records' (rows), whereas file systems optimize for smaller numbers of relatively larger 'files.' Plenty of overlap.
>
> Although file systems are not tied to b-trees as closely as databases usually are, many modern file systems use b-trees ubiquitously (such as btrfs on Linux, ReFS on Windows?, AFS on macOS? Others worth mentinoinng?)

### Handling Large Items

> How does a file system handle these large items? You can't stick them in your b-tree because they're often much larger than any particular node of your b-tree.
>
> Think about the interface a disk provides, and fundamentally how a b-tree is laid out on disk
>
> Diagram showing a byte array, b-tree nodes in that byte array pointing to each other
>
> Idea: drop data anywhere on disk you have room to put it, then use a b-tree record to point to the data:
>
> Diagram

### Naming Hierarchy

> Very quick overview of files and directories. Show how a b-tree is a good way to handle this

### File Contents

> Show how to use a b-tree to index over a file's contents, which are stored as blocks elsewhere on disk

### Disk Allocation

> Explore the problem of allocating space on a disk, both for allocating blocks of space for file contents and for allocating blocks of space for b-tree nodes. Note that a b-tree can even be used for this.
>
> Mention fragmentation, this is more than just a data structures problem, very similar to writing a memory allocator since disks are very similar to memory

## When to Build Storage Systems

> Salvage some of the content from the old 'optimizing and tradeoffs' section. The main argument is that every system optimizes for certain workloads and the expense of being worse at other workloads, and sometimes you have a workload that nobody has ever optimized for &mdash; and that's when you break out this theory and go build your own system.

## Where To Next

> Other concerns like resiliency (checksums), concurrency control
>
> New unconventional types of storage devices which meld performance aspects of disks and memory
>
> Append-only stores and the unique challenges they pose
>
> Distributed stores and the unique challenges they pose

## Additional Reading

> Plug DDIA

---

> This originally appeared after the 'key problem' section and before launching the discussion about hash tables, binary search and so on. But it made the flow kind of weird, and I'm not positive we need to build up this terminology as part of the discussion.
>
> Cut if we manage to make it through the rest of the article without using these terms.

> ## Optimizing and Tradeoffs
>
> When you're designing an index, you want your write algorithm to (efficiently) set up for reads (to also be efficient). But what does "efficient" mean in this context?
>
> It might mean that read and write requests complete with low **latency**. If we were being pedantic, "latency" has a very specific meaning to storage people, but colloquially speaking, latency usually means te time between two events. In storage systems, we'll mostly be interested in request latency: how long it takes to execute a request.
>
> "Efficient" might also refer to how much **throughput** the system provides. Throughput is kind of the inverse of latency: in a given amount of time, how much can the system do? You might measure throughput as a number of requests executed per second (often labeled "I/O Operations Per Second" or "IOPS") or as the number of bytes transferred to/from the client per second.
>
> In practice, latency and throughput are often related. As a system nears its throughput limits, request latency tends to go up, because when a system can't keep up with the requested load, requests end up often needing to wait for other requests to progress, driving up latency for all requests. These waits are called *queuing delays*.
>
> So from a user's perspective, "efficient" usually means providing reasonably low latency at a reasonably high throughput.
>
> We, the designers of the storage system, usually start out by defining a target latency at a target throughput and optimize from there. In optimizing our map data structure, we have complete control over our map data structure and algorithms, but no control over the client, who can issue any pattern of read and write requests at any rate, often concurrently.
>
> As we'll soon see, there are a lot of tricks we can use to optimize our data structures and algorithms, but most optimizations are tradeoffs that make the system better at certain request patterns at the expense of being worse at other patterns. That has a couple of important ramifications:
>
> As storage system designers, we need to make assumptions about what kinds of request patterns clients will use the system for, so we can make tradeoffs that improve the common case while making the system worse at things that happen rarely or never.
>
> As users of storage systems, we need to select a system that is optimized for the request patterns we intend to use. On rare occasions, we might find there is no system optimized for our intended workload: that's when it's time to put on our system designer hats and roll our own!