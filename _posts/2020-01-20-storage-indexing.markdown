---
layout: post
title: Storage (for Mortals)
author: Dave
draft: true
---

*Never roll your own database!*

Most programmers agree with this advice. But have you ever stopped to consider why not? It can't be that *nobody* should build new file systems, databases, distributed search indexes and so on &mdash; times change, people change, hardware changes, and we aren't still using the original systems we designed back in the 1960s-70s. Clearly, some people must be out there successfully building their own data storage systems; so why shouldn't you or I?

One fine answer is time: as a rule of thumb, it takes about 10 years to fully stabilize a new general-purpose data storage system to the point of being near-optimal and also bug-free, and for most projects you don't have that kind of time. There are many off-the-shelf databases, file systems and so on available to you, and chances are getting one of them to work reasonably well in your application will take less than the time it'd take to build and stabilize one of these systems yourself.

But you certainly can build one of these systems yourself, if you want. It could be you'd like to get a better feel for how these systems work internally, or maybe you want to see if it's fun. There are people who will tell you that these systems are too complicated for us mortals to understand, and that such things are best left to the *Experts* (TM). In this article, I want to show this isn't true &mdash; it's not too hard to come up with the basic design of these systems as long as you have a good grasp of basic data structures and algorithms.

Let's try walking down a fairly simple line of reasoning that gets you more or less to today's state of the art in data storage. All you'll need is some basic familiarity with data structures like hash tables and binary trees.

Without further ado ...

## Let's Design a Store

We'll start with one of the most basic kinds of storage systems: a key/value store, where we store both the keys and values on disk. We'll provide the following basic interface:

* The client (our user) defines the structure of a **key** and a **value**
* We provide a **write** method which accepts a key and a value, and adds them to our on-disk store
* We provide a **read** method which accepts a key, and reads the corresponding value from disk

This interface should seem familiar: it's the basic key/value map interface provided by many programming languages' standard libraries and runtimes. If this seems like an odd interface to use for disk storage, keep in mind that most "real" storage systems rely on something that looks an awful lot like a key/value mapping, even if these systems don't specifically use the words *key* and *value*. For example:

* Want to download a web page? You need to provide its URL (a key)
* Want to read a file? You need to provide its path (a key)
* Want to load a value from memory? You need to provide its address (a key)
* Want to query a database? You need to provide a primary *key*

Later on we'll see how to gussie up an on-disk key/value store like this one to make it the core of a database, or a file system. But for now let's focus on how to build a store like this one efficiently.

## The Key Problem

When designing something, a good question to answer first is: what's the hard part? What should we spend most of our time thinking about how to solve?

Ironically, the 'storing the data' part of writing a data store isn't all that hard. The hardware does all the work, so from a software perspective all we really need to do is set up the transfer and get out of the way.

The hard part is on the other end: retrieval. You have a disk (maybe a lot of disks) full of data, maybe written incrementally by lots of users over a long time. Today, a user comes along and requests a specific handful of bytes stored somewhere along your sea of billions (of bytes). How do you efficiently find the specific data the client requested?

In other words, storage is a search problem! <span style="font-size:85%">(Also, the title of this section is a bad pun!)</span>

So in designing our key/value store, the main challenge to overcome is designing an efficient on-disk key/value map data structure, where the write method efficiently sets up for reads to also be efficient.

## Map Data Strutures

Now that we know the goal, let's start looking at options. We want a key/value mapping data structure to store our data on disk &mdash; what are some map data structures we already know?

### Hash Tables

Hash tables (also known as hash maps and dictionaries) are easily the most popular data stuctures for mapping keys to values. Chances are, when you need a data structure that maps keys to values, you're going to reach for a hash table without really thinking about alternatives; after all, given hash tables provide constant time insert and lookup on average, why bother with anything else?

The basic idea of a hash table is to store key/value pairs in an array, and use a hash function to map each possible key to an index in this array:

> Diagram

Since both the hash function and array access run in constant time, the hash table is also constant time overall.

Or is it? The main challenge with hash tables is dealing with *collisions*: cases where two different happen to get mapped to the same array index:

> Diagram

This problem is unavoidable. For many commonly used key types (like ints and strings), there are many, many possible keys (possibly even more than there are atoms in the known uinverse), and you only have so many array slots to hold these keys in. There must be cases where two keys map to the same array slot simply because there simply aren't enough array slots for each key to get its own slot (this is sometimes called the [pigeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle)).

Hash tables are still an area of active academic research, and that research is focused on finding better ways to avoid hash collisions, and to deal with the ones you've ended up with. [Wikipedia](https://en.wikipedia.org/wiki/Hash_table) has a pretty good discussion of the many different approachs people have come up with.

But even with all this research, every known strategy for handling collisions takes $O(N)$ time in the worst case, meaning in the worst case for a hash table, the time it takes to insert one key/value mapping is proportional to the number of key/value mappings already in the hash table.

### Binary Search Trees

Although they're less commonly known and used, binary search trees are another perfectly valid way to build and search a key/value mapping.

The basic idea is to construct a tree, where each node has a key/value pair as well as pointers to two child nodes. These nodes are sorted: given a node, all nodes that can be reached from the current node's left child pointer have keys that are 'less' than the current node's key, and similarly, all nodes reachable from the right child have keys 'greater' than the current node's key:

> Diagram

It can be shown that, with this data structure, it takes $O(\log{N})$ time to insert a key/value pair or to find a node with a given key. Even if this isn't technically constant time, for most data sets this is so fast that it's barely distinguisable from constant time; and unlike hash tables being 'usually' constant time, $O(\log{N})$ is the true worst case for a binary search tree: a binary search will never take $O(N)$ time in any circumstance.

> Should we introduce range queries?

## Maps on Disks

Do people use data structures like hash tables and binary search trees to store and retrieve data on a disk?

You certainly can: at the lowest level, a disk is a hardware device that exposes data storage as an array of bytes. Computer memory (RAM) is also exposed as an array of bytes, so anything you can store in memory you can also store on disk in the exact same way. The main functional difference between memory and disk storage is that memory gets wiped when the system powers off, whereas data on a disk is *persistent* (rebooting the computer, unplugging it and so on doesn't change the data).

People do sometimes build hash tables and binary search trees for data on disk, but it isn't so common. Despite the similarities between memory and disks, there's one big catch: they perform very differently!

### Disk Performance

The different parts of your computer don't run at the same speed, and reading and writing disks are some of the slowest things your can do. The following table gives a rough idea of just how much slower your disk is than the rest of your computer:

| **Device**        | **Latency (Actual)** | **Latency (Human Scale)** |
| ----------------- | -------------------- | ------------------------- |
| CPU               | 0.4 ns               | 1 sec                     |
| Memory            | 100 ns               | 4 min                     |
| Disk (SSD)        | 50-150 μs            | 1.5-4 days                |
| Disk (Rotational) | 1-10 ms              | 1-9 months                |
| Network Request   | 65-141 ms            | 5-11 years                |

*(Hat tip to [Prowess](https://www.prowesscorp.com/computer-latency-at-a-human-scale/) for the excellent visualization &mdash; which I have copied here shamelessly)*

But even this isn't the full story: Take a look at these results from a little performance test I ran on my computer's main disk (an SSD):

![Disk Access Latency vs I/O size graph](/assets/disk-access-latency.png)

I generated these graphs by repeatedly executing disk reads and writes, measuring the latency (how long it took) for each, then averaging the results together. The graph shows these average latencies as a function of the number of bytes transferred, so the X axis (horizontal) is the number of bytes transferred in each disk I/O operation, and the Y axis (vertical) is how long it took to transfer that many bytes on average..

Conventional wisdom goes that the time it takes to read or write data on a disk is proportional to the amount of data you read or write. Since the X axis of each graph ("bytes transferred") increases at an exponential rate, doubling progressively at each point, this should imply the graph of the latency vs bytes transferred rises exponentially ...

... and, eventually, it does. But look at the left side of each graph: for smallish I/O sizes of about 64 KB or less, both graphs are roughly flat. This means something a little weird: the time it to read or write 1 KB of data was about the same as it was to read or write about 32 KB of data. Weird, right?

(The main reason this happens is that the disk is a separate device inside your computer, and there's work that has to be done to have the CPU send a request to a disk, the disk to enqueue and process the request, and so on; this work is the same no matter how many bytes you're transferring, but only for larger I/O sizes does the time taken to actually read/write the data start to dominate this fixed per-request overhead cost.)

Now that we have an idea of how disks perform, let's try to design a key/value map data structure that takes advantage of the good parts and avoids pitfalls.

### Caching

The first basic observation we can make by looking at the table above is that reading from memory is faster than reading from a disk, so if we want queries to our key/value map to be fast, we should store the map in memory.

Of course, things are never so simple. One problem is that we need our key/value map to persist data across reboots and power loss, but memory isn't persistent. Another  is that memory is much more expensive per megabyte than disks are, so we will almost always have way less memory than we have disk storage, so even if we use all of the computer's memory we might not have enough room to store all the data we need.

A simple solution, then, is to selectively **cache** parts of our key/value map in memory. That is, the entire key/value map is stored on disk persistently, but we also store a copy of parts of the map in memory. If someone requests a part of the map that we happen to have in our in-memory cache, we can query the in-memory mappings and skip all disk I/O, saving a bunch of time. Nice!

One of the tricky parts of designing a cache is the cache's policy: the rules you choose for determining which parts of your map are worth storing in memory and which are not. Most cache policies are built around two key observations:

* **Spatial locality of reference**: client likely to visit key ranges near key ranges it previously queried
* **Temporal locality of reference**: client likely to revisit key ranges it recently queried

Putting these together and dropping the academic-speak, the basic suggestion is as follows: when a client reads a key from the mapping, store a copy of that key/value pair as well as nearby key/value pairs in memory, in case the client comes back for the same key or a nearby key soon.

Looking back at hash tables and binary search trees, we immediately see a problem with caching data for hash tables: if we want to exploit spatial locality, we need a way to find and read in all keys that are 'near' the key the client just requested; but there isn't a good way to do that with a hash table, because hash tables *deliberately* scramble keys to minimize the likelihood of a hash collision:

> Diagram: `A B C D E -> 3 1 5 4 2 -> C A E D B`

No bueno.

Binary search trees are a little better in this regard, because they sort entries by key.







---



### Access Times and I/O Sizes

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

### Naming Hierarchy

> Very quick overview of files and directories. Show how this is basically the database problem all over again, and how b-trees are a good fit

## File Contents and Handling Large Items

> The database design assumed many small key and value records, but files can get pretty darn big: the name is still a small key, but the value (file contents) can be gigabytes to terabytes in size. You can't stick these file contents in your b-tree because they're often much larger than any particular node of your b-tree.
>
> Do we need to throw away our system and start over with something new? How does a file system handle these large items? 
>
> Think about the interface a disk provides, and fundamentally how a b-tree is laid out on disk
>
> Diagram showing a byte array, b-tree nodes in that byte array pointing to each other
>
> Idea: drop data anywhere on disk you have room to put it, then use a b-tree record to point to the data by its disk address:
>
> Diagram

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