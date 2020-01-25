---
layout: post
title: Storage, for Mortals
author: Dave
draft: true
---

*Never roll your own database* is good advice, but one downside of telling people never to roll their own storage systems is people coming to think they shouldn't roll their own because they can't &mdash; that there must be something mysterious and arcane about building these systems that only a few select individuals could ever hope to build one.

Lucky for us, that's not true!

Databases, file systems and the like are designed by engineers just like you and me, using concepts you've probably already learned in a college-level introductory course on data structures and algorithms. My goal with this blog is to convince you of this by showing you the basic chain of reasoning that leads you to more or less the state of the art in databases, file systems and so on.

If these systems are simple enough we can cover them in a blog post, you might ask, why not roll your own The main answer is time: a common rule of thumb is that is takes 10 years to stabilize a file system or database such that it's optimally fast and also bug-free. But the hard part isn't so much the high level design as the low-level implementation; mostly what makes storage hard is that multithreading is hard.

So, let's design some storage systems!

## The Key Problem

Beware: the title of this section is a (dumb) pun.

Ironically, the hard part about designing a sotrage system isn't storing data; hardware does a pretty fine job of that already, and in software, our main job is to funnel the data to the hardware and not get in the way.

The hard part is retrieval &mdash; getting the data out of hardware is just about as easy as getting the data into hardware in the first places, but first *finding* the data the user asked for is hard.

In other words, storage is a search problem.

In pretty much every storage system that has ever existed, all stored data is identified by some sort of key: the client presents a key to the storage system, the storage system does 'something' to figure out where in hardware the data is being stored, and the data asks the hardware to retrieve the data so it can be returned to a client. The 'something' is essentially a search over a key/value mapping.

A lot of systems disguise the key, but if you look closely enough at a system that stores and retrieves data, you'll usually find something that looks a lot like a key:

* Want to load a web page? You need to provide its URL (a key)
* Want to read a file? You need to provide the file's path (a key)
* Want to read from memory? You need to provide an address (a key)
* Want to read from a database? You need to provide a primary key (... obviously a key!)

In all of these systems, the key (or key-like thing) is an identifier for one and only one resource/record/etc. Each of these systems provides a query operation that, given a key, returns the resouce identified by that key.

Keys pop up in all storage systems because you can't deal with all the data all the time (if you could, you wouldn't need the storage system in the first place). Keys allow the system's user to 'amplify' knowledge: if the client knows a small piece of information about something (a key), the client can provide it to the storage system to learn more about that thing (a record). This record might have additional keys the client can query the storage system about to learn more about related things.

So storage is basically a search problem over a key-value map data structure. Every storage system provides a write operation, which puts a new key/value pair in this map, and a read operation, which given a key returns the corresponding value by searching this map. The write algorithm aims to (efficiently) set up for reads (to also be efficient).

## Optimizing and Tradeoffs

But what does "efficient" mean?

It might mean that read and write requests complete with low **latency**. Latency colloquially refers to the time that elapses between two events. In storage, those two events are usually a request starting and that request completing: so a request latency measures how long the system took to execute the request.

Or, "efficient" might refer to how much **throughput** the system provides. Throughput is kind of the inverse of latency: in a given amount of time, how much can the system do? You might measure throughput as a number of requests executed per second (often labeled "I/O Operations Per Second" or "IOPS") or as the number of bytes transferred to/from the client per second.

In practice, "efficient" usually means a combination of these things. As a system nears its throughput limits, request latency tends to go up, because a system nearing its throughput limits must queue some requests, each request waiting for some other requests to complete, driving up latency. So from a user's perspective, "efficient" usually means providing reasonably low latency at the level of throughput the user intends.

We, the designers of the storage system, usually start out by defining a target latency at a target throughput and optimize from there. We have complete control over our map data structure and algorithms, but no control over the client, who can issue any pattern of read and write requests at any rate, often concurrently.

As we'll soon see, there are a lot of tricks we can use to optimize our data structures and algorithms, but most optimizations are tradeoffs that make the system good at certain request patterns at the expense of being worse at other patterns. For storage system designers, this means making assumptions about how the client will use the system; for clients, this means choosing a system that optimizes for the request pattern the client intends to use. And sometimes, there is no system optimized for the client's workload: that's when it's time to roll your own storage system!

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