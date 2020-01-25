---
layout: post
title: Storage, for Mortals
author: Dave
draft: true
---

*Never roll your own database* is good advice, but never let that convince you that you *can't* roll your own! Databases, file systems and the like are designed by engineers just like you and me, using concepts you've probably already learned in an introductory class on data structures and algorithms. If you have a grasp of these basic concepts, you're already well on your way to knowing how to build one of these systems on your own.

The reason people tell you not to roll your own database, file system, etc mostly boils down to time: a common rule of thumb is that is takes 10 years to stabilize a file system or database to the point of being near-optimally fast and also bug-free. Most of those 10 years will be spent on low-level implementation concerns, primarily dealing with multithreading being hard.

But the high level design of these systems is comparatively easy. My goal with this article is to walk you down the line of reasoning that leads you close to the state of the art in databases, file systems and so on. All you'll need is some familiarity basic data structures like hash tables and binary trees.

So, let's design some storage systems!

## The Key Problem

What do you think is the main challenge of designing a storage system?

Hint: it's (ironically) not the part where you actually store the data. That's pretty easy from a software standpoint: hardware does a pretty fine job of storing data, and as software developers we mostly just need to funnel the data to the hardware and not get in the way.

Other than storing data, there's really only one other thing a storage system has to do &mdash; retrieve data &mdash; so the main challenge must lie somewhere in there. Let's take a closer look at retrieval.

In pretty much every storage system that has ever existed, all stored data is identified by some sort of key: the client presents a key to the storage system, the system does 'something' to figure out where in hardware the data is being stored, and the system asks the hardware to retrieve the data so it can be returned to a client.

A lot of systems disguise the key, but if you look closely enough at a system that retrieves data, you'll usually find something that looks an awful lot like a key:

* Want to load a web page? You need to provide its URL (a key)
* Want to read a file? You need to provide the file's path (a key)
* Want a value from RAM? You need to provide an address (a key)
* Want to query a database? You need to provide a primary key (... which is obviously a key!)

In all of these systems, the key (or key-like thing) is an identifier for one and only one resource/record/etc. Each of these systems provides a query operation that, given a key, returns the resouce identified by that key.

> You might wonder why this is so commonly true. Keys pop up in storage systems because you can't deal with all the data all the time &mdash; if you could, you wouldn't need that storage system in the first place!
>
> Keys allow a client to 'amplify' knowledge: if the client knows a small piece of information about something (a key), the client can provide it to the storage system to learn more about that thing (a record). This record might have additional keys the client can query the storage system about to learn more about related things.

So, back to the original question: what makes retrieval challenging? The answer is the 'something' we mentioned the system has to do to find the data for a given key. How does a system find the data for a client-provided key?

In other words, storage is a search problem!

So the key problem is the problems with keys: given a key, how do we figure out where we previously stored the corresponding data? The answer is we need to store a map data structure that maps every key to the corresponding data in storage hardware. Our system's write algorithm aims to (efficiently) set up for reads (to be efficient).

## Optimizing and Tradeoffs

But what does "efficient" mean?

It might mean that read and write requests complete with low **latency**. If we were being pedantic, "latency" has a very specific meaning to storage people, but colloquially speaking, people often use the term to refer to the time between two events. In storage systems, those two events are usually a request starting and that request completing: so a request's latency is how long the system took to execute that request.

Alternately, "efficient" might refer to how much **throughput** the system provides. Throughput is kind of the inverse of latency: in a given amount of time, how much can the system do? You might measure throughput as a number of requests executed per second (often labeled "I/O Operations Per Second" or "IOPS") or as the number of bytes transferred to/from the client per second.

In practice, latency and throughput are often related. As a system nears its throughput limits, request latency tends to go up, because when a system can't keep up with the requested load, requests end up often needing to wait for other requests to progress, driving up latency for all requests. These waits are called *queuing delays*.

So from a user's perspective, "efficient" usually means providing reasonably low latency at a reasonably high throughput.

We, the designers of the storage system, usually start out by defining a target latency at a target throughput and optimize from there. In optimizing our map data structure, we have complete control over our map data structure and algorithms, but no control over the client, who can issue any pattern of read and write requests at any rate, often concurrently.

As we'll soon see, there are a lot of tricks we can use to optimize our data structures and algorithms, but most optimizations are tradeoffs that make the system better at certain request patterns at the expense of being worse at other patterns. That has a couple of important ramifications:

As storage system designers, we need to make assumptions about what kinds of request patterns clients will use the system for, so we can make tradeoffs that improve the common case while making the system worse at things that happen rarely or never.

As users of storage systems, we need to select a system that is optimized for the request patterns we intend to use. On rare occasions, we might find there is no system optimized for our intended workload: that's when it's time to put on our system designer hats and roll our own!

## Map Data Strutures

So now we know a storage system is a key-value map data structure stored on disk. What are some map structures we already know?

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