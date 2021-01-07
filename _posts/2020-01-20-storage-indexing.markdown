---
layout: post
title: Modern Storage Systems
author: Dave
draft: true
---

Did you know there's a small set of basic data structures and algorithms that underpin just about every system that stores and retrieves data on disk, from databases to file systems and more?

It turns out there's a fairly straightforward line of reasoning that you can follow to get more or less to today's state of the art &mdash; all you need is some basic familiarity with data structures like hash tables and binary trees:

## Let's Design a Store

We'll start with one of the most basic kinds of storage systems: a key/value store, where we store both the keys and values on disk. We'll provide the following basic interface:

* The client (our user) defines the structure of a **key** and a **value**
* Our **write** method adds a key/value pair to our store
* Our **read** method accepts a key, and reads the corresponding value from the store

This interface should seem familiar: it's the basic key/value map interface provided by many programming languages and runtimes. If this seems like an odd interface to use for disk storage, keep in mind that most "real" storage systems rely on something that looks an awful lot like a key/value mapping, even if these systems don't specifically use the words *key* and *value*. For example:

* Want to download a web page? You need to provide its URL (a key)
* Want to read a file? You need to provide its path (a key)
* Want to load a value from memory? You need to provide its address (a key)
* Want to query a database? You need to provide a primary *key*

Later on we'll see how to gussy up an on-disk key/value store like this one to make it the core of a database, or a file system. But for now let's focus on how to build a store like this one efficiently.

## The Key Problem

When designing something, a good question to answer first is: what's the hard part? What should we spend most of our time thinking about how to solve?

Ironically, the 'storing the data' part of implementing a data store isn't all too challenging. The hardware does all the work, so from a software perspective all we really need to do is set up the transfer and get out of the way.

The hard part is on the other end: retrieval. You have a disk (maybe a lot of disks) full of data, maybe written incrementally by lots of users over a long time. Today, a user comes along and requests a specific handful of bytes rattling around somewhere in there. How do you efficiently find the specific data the client requested?

In other words, storage is a search problem! <span style="font-size:85%">(Also, the title of this section is a bad pun!)</span>

So in designing our key/value store, the main challenge to overcome is designing an efficient on-disk key/value map data structure, where the write method efficiently sets up for reads to also be efficient.

## Map Data Strutures

Now that we know the goal, let's start looking at our options. We want a key/value mapping data structure to store our data on disk &mdash; what are some map data structures we already know?

### Hash Tables

Hash tables (also known as hash maps and dictionaries) are easily the most popular data stuctures for mapping keys to values. Most developers instinctively reach for one of these whenever they need a map, without considering alternatives; after all, hash tables provide constant time insert and lookup, so why bother with anything else?

The basic idea of a hash table is to store key/value pairs in an array, and use a hash function to map each possible key to an index in this array:

> Diagram

Since both the hash function and array access run in constant time, the hash table is also constant time overall.

Or is it? The main gotcha with hash tables is dealing with *hash collisions*: cases where two different keys happen to get mapped to the same array index:

> Diagram

This problem is hard to avoid: whenever there are more possible keys than there are array slots, there have to be cases where the hash function maps two keys to the same slot, because there simply aren't enough array slots for each key to gets its own (this is sometimes called the [pigeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle)). There's still ongoing academic research into finding better ways to avoid collisions, as well as better ways to deal with them ([Wikipedia](https://en.wikipedia.org/wiki/Hash_table) has a pretty good discussion of the approaches people have come up with).

But even with all this research, every known strategy for handling collisions takes $O(N)$ time in the worst case, meaning in the worst case for a hash table, the time it takes to insert one key/value mapping is proportional to the number of key/value mappings already in the hash table. However, in the absence of collisions, hash table operations are $O(1)$, so hash tables are often said to be "constant time in the average case." In most cases, our hash functions and collision handling strategies work pretty well, so this assumption tends to hold.

### Binary Search Trees

Although they're less common, binary search trees are another perfectly valid way to build and search a key/value mapping.

The basic idea is to construct a tree, where each node has a key/value pair as well as pointers to two child nodes. These nodes are sorted: given a node, all nodes that can be reached from the current node's left child pointer have keys that are 'less' than the current node's key, and all nodes reachable from the right child have keys 'greater' than the current node's key:

> Diagram

It can be shown that, with this data structure, it takes $O(\log{N})$ time to insert a key/value pair or to find a node with a given key. Even if this isn't technically constant time, the $\log$ function grows very slowly, so for most datasets this is close enough to constant time that the difference is barely discernable. Plus, for binary search trees, $O(\log N)$ is truly the worst case: a binary tree operation will never take $O(N)$ time (like a hash table can in the event you get a lot of collisions).

## Maps on Disks

The logical next question is: can we use these to implement our key/value store? Are these data structures good choices?

You certainly *can* use hash tables or binary trees as the data structure for an on-disk key/value store: at the lowest level, disks expose an array of bytes just like memory does, so anything you can store in memory you can ultimately store on a disk in the exact same way. The only major difference between memory and disk interfaces is that memory gets wiped when the system powers off, whereas data on a disk is *persistent* across power cycles.

So we can build a hash table or a binary tree on disk; but should we? People sometimes do, but it isn't so common. Despite the similarities between memory and disks, there's one big catch: they perform very differently!

### Disk Performance

The different parts of your computer don't run at the same speed. The following table gives a rough idea of just how much slower your disk is than the rest of your computer:

| **Device**        | **Latency (Actual)** | **Latency (Human Scale)** |
| ----------------- | -------------------- | ------------------------- |
| CPU               | 0.4 ns               | 1 sec                     |
| Memory            | 100 ns               | 4 min                     |
| Disk (SSD)        | 50-150 μs            | 1.5-4 days                |
| Disk (Rotational) | 1-10 ms              | 1-9 months                |
| Network Request   | 65-141 ms            | 5-11 years                |

<center><i>(Hat tip to <a href="https://www.prowesscorp.com/computer-latency-at-a-human-scale/">Prowess</a> for this visualization)</i></center>

As you can see, readng and writing data on a disk is one of the slower things your code can do: even with a fast SSD, accesing the same data in memory is hundreds of times faster!

But even that isn't the full story: Take a look at these results from a little performance test I ran on my computer's main disk (an SSD):

![Disk Access Latency vs I/O size graph](/assets/disk-access-latency.png)

For this test, I executed many reads and writes from my disk using different I/O sizes (the number of bytes transferred per request). In the graphs above, the horizontal axis shows this I/O size, and the vertical axis shows the average latency (time to execute the transfer) for each I/O size in milliseconds.

Conventional wisdom says the time it takes to read or write data is proportional to how much data you want to read or write; if this were true, we would expect to see the latency grow exponentially in these graphs, because the horizontal axis grows exponentially (it doubles at each point).

On the right hand side of our graphs, we do eventually see this exponential curve, but notice the left sides are roughly flat. This means, on average, the time it took to transfer 1 KB of data is about the same as it took to transfer 32 KB of data. (Why? This happens because there's a fixed cost of setting up a disk I/O operation, which dominates the cost of actually reading/writing data unless you're trying to read or write a lot of data.)

With these characteristics in mind, let's re-evaluate or two map data structures.

### Caching

The first basic observation we can make by looking at the table above is that reading from memory is faster than reading from a disk, so if we want queries to our key/value map to be fast, we should store the map in memory.

Well, things are never so simple. One problem is that we need our key/value map to persist data across reboots and power loss, but memory isn't persistent. Another is that memory is much more expensive per megabyte than disks are, so we will almost always have way less memory than we have disk storage; even if we use all the computer's memory, we might not have enough room to store all the data we need.

A simple solution, then, is to selectively **cache** parts of our key/value map in memory. That is, the entire key/value map is stored on disk, persistently, but we also store copies of parts of the map in memory. If someone requests a part of the map that we happen to have in our in-memory cache, we can query the cached copy and skip the disk I/O entirely, saving a bunch of time. Nice!

Caches like these generally take advantage of two common observations about how users use storage systems:

* **Spatial locality**: clients are likely to query key ranges nearby key ranges they previously queried
* **Temporal locality**: clients are likely to revisit key ranges they recently queried

Putting these together and dropping the academic-speak, the basic suggestion is: when a client reads a key from the mapping, store a copy of that key/value pair as well as nearby key/value pairs in memory, in case the client comes back for the same key or a nearby key soon.

How easy is it to implement a cache like this on top of our two candidate map data structures?

For hash tables, we immediately see a problem: if we want to exploit spatial locality, we need an easy way to find all keys that are 'near' the key the client just requested; but there isn't a good way to do that with a hash table, because hash tables *deliberately* scramble keys to minimize the likelihood of a hash collision:

> Diagram: `A B C D E -> 3 1 5 4 2 -> C A E D B`

No bueno.

Binary search trees are a little better in this regard, because they sort entries by key; if you're already looking up a key, it's not too hard to find neighboring keys because the tree stores the keys in sorted order:

> Diagram showing a sorted tree, where we read in a subtree in the cache and then return the requested key to a client

Score one for binary trees.

### Access Counts

Let's take another look at the latency table from earlier, but this time let's use the latencies to figure out how many 'things' we can do with each device per second:

| **Device**        | **Latency (Actual)** | Operations / Second |
| ----------------- | -------------------- | ------------------- |
| CPU               | 0.4 ns               | 2,500,000,000       |
| Memory            | 100 ns               | 10,000,000          |
| Disk (SSD)        | 50-150 μs            | 6,700-20,000        |
| Disk (Rotational) | 1-10 ms              | 100-1,000           |
| Network Request   | 65-141 ms            | 7-16                |

(All I did here was invert each latency &mdash; for example, there are 2.5 billion 0.4-ns intervals per second.)

The key observation I want you to draw from this table is that with a disk (especially rotational disks), you don't get all too many operations per second to work with. For our key/value store, we want to minimize the number of disk accesses per client request, because each disk access is already pretty costly. Do either hash tables or binary search trees do a good job of that?

For hash tables, the answer is a little tricky. When the client gives you a key, you would hash it to figure out where on disk the value should be, and go read that. If that was the value you wanted, you're done in just one disk access; but if there was a hash collision, you have to do something to go find the next key. In general, the number of reads needed is the number of collisions which exist for the given key &mdash; and in the worst case, you could have to read every single byte of data on your disk to find the value being requested!

For binary search trees, the answer is more straightforward, but not what we want. It takes $\log_2 N$ time to search a binary search tree, so if you have a modestly large store with a million key/value pairs, you'll need to follow 20 nodes to get all the way from the root of the tree to a leaf. That's pretty bad: if you're using a disk that can do 1,000 reads per second, but you have to do about 20 reads per client query, you can only answer 50 client queries per second. Not very scalable at all!

It would seem hash tables are sort of the winner here, but neither data structure is ideal.

### Our Wish List

It's starting to look like neither a hash table or a binary search tree is a good data structure for implementing a key/value store, because neither performs particularly well on disk hardware.

What are some things we'd ideally like in a data structure for a key/value map on disk?

First, we want to keep keys in sorted order. That way we can easily implement a cache of our data structure which exploits both spatial and temporal locality of reference.

Second, in cases when the key being requested is not cached, we want to minimize the 'worst case' number of disk reads needed to search the data structure.

Finally, we also want to store key/value pairs in a way that makes them easy to read in batch: after all, it takes roughly the same amount of time to read 32 bytes or 32 *kilo*bytes from the same disk, so we might as well read a little extra and cache it in case the client requests it soon.

There's a data structure that fits this bill: the **b-tree**.

## B-Trees

Intuitively, the basic idea of a b-tree is to start with a binary search tree and 'smoosh' it down, kind of like this:

> Diagram

Another way to think of this is making a binary tree 'wider' &mdash; whereas a binary tree has two children per node, a b-tree is an '$N$-ary' tree with $N$ children per node (where $N$ is often in the hundreds). Typically, we pick a b-tree node size between 8 and 64 kilobytes, and stuff in as many key/value/child pairs as will fit.

To search a b-tree, we first read the root node from disk. We then search that node's items array looking for either the item we want, or a gap between two items where our item would appear. In the latter case, we follow the child pointer to another node, which we then read from disk and search in the same way, recursively:

> Diagram

We're fundamentally still binary-searching a sorted collection as a tree, so the search complexity of our map is still $O(\log N$). However, since there are hundreds of items per node, there are hundreds times fewer nodes in a b-tree than there would be in a binary search tree with the same items. This saves us a lot of disk reads during searches.

Constructing a b-tree is a little more involved, but nothing we can't handle. Wikipedia has [a pretty good writeup](https://en.wikipedia.org/wiki/B-tree) if you're curious about learning more.

### Nice Properties of B-Trees

Compared to binary search trees, what nice properties do we get by 'smooshing' the tree down into a wider, shallower structure with large nodes?

The main advantage is making it almost trivial to build an in-memory cache that exploits both spatial and temporal locality. All we need to do is cache a copy of each b-tree node we've accessed recently: if the client comes queries a nearby key soon, chances are very good that key is in the b-tree node we just cached, so we can serve the query from the cached node with no disk reads.

Even better, reading a b-tree's large nodes is 'free' compared to reading a binary search tree's very small nodes, because the time it takes to read a handful of bytes from a disk is the same as it takes to read a handful of kilobytes &mdash; remember these graphs?

![Disk Access Latency vs I/O size graph](/assets/disk-access-latency.png)

Finally, even if we don't have the requested data in our cache, the very high branching factor of a b-tree keeps it shallow; for example, if you had a million items in a b-tree with 256 items per node, your tree would only have three levels, which means even if your cache were empty, you would only need to execute 3 disk reads to serve a client query (compared to 20 for binary trees, as we estimated earlier).

In practice, though, you'd basically never execute all three of those disk reads, because there's an extremely good chance you cached some of the internal nodes already. For example, as long as you've queried *any part* of the tree recently, you already have the root node cached; that's one less disk read you need.

### The Ubiquitous B-Tree

For disk query performance, it's hard to beat a b-tree, which is why they're used across all sorts of data stores, from file systems to databases to caches and more. Much work has gone into optimizing b-trees, and there are many variants (the [Wikipedia article on b-trees](https://en.wikipedia.org/wiki/B-tree#Variants) includes a nice overview of the different variants people have come up with).

But, for now, let's get back to our key/value store.

## Let's Build a Key-Value Store

Now that we have a good data structure in mind, let's build a key/value store on disk using a b-tree, and analyze its performance.

> Complete the key-value store interface on top of this tree. Should be very easy.

## Let's Build a Database

> This was originally written before we refactored the post to describe a generic 'key/value store' first, so this section may need some redrafting.

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
