---
layout: post
title: Log-Structured Merge
author: Dave
draft: true
---

## Newer brain dump

I don’t love the name log structured merge tree, in large part because this obscures a more general pattern that’s not tree specific. You can really use any data structure. Instead I will call this Overlay/Merge (paradigm, pattern, what have you)

We can derive this from needs and first principles

Say I have a data structure, can be anything you like
Say the data structure is too large to fit in memory, so I have to save it to disk
Clearly I should not always go direct to disk, that’d always be slow, so instead I cache recent reads and buffer recent writes in memory
If you really hammer this system with randomized reads, the cache will probably do poorly and I’ll need to go to disk a lot
If you really hammer this system with writes, my memory buffer will fill up and I’ll be forced to stall while writing it out to disk, no gains from buffering in memory
But hey if the load isn’t crazy I get the latency benefits of using RAM

Let’s say I’m really going to hammer this system with writes, and read very infrequently. Can I do better than the approach above?
Insight: not all writes are created equal. Editing the on disk data structure will require multiple separate randomized reads and writes, but a large sequential read of the same size will take much much less time
Idea: if the memory buffer gets full, bail the whole thing out to disk as one big sequential write, and in the meantime use the reclaimed memory to continue accepting writes
In a system like this, reads work as follows: first see if the memory buffer contains the data I’m looking for. If no, go to the most recent buffer we dumped to disk. If it’s not there either, go to the second most recent, and so on.

This is the “overlay” of overlay/merge. The memory buffer, and each on disk buffer is an “overlay” of the data structure.

Now an obvious problem is as we accept more writes, reads take longer. The runtime of a read grows as O(writes) because we have to check that many overlays as time goes on.
The obvious solution is to merge overlays together, coalescing them all together over time. This is exactly the slow thing we were trying to avoid, but it happens off the “hot path” so write throughout is better than if we always had to merge when flushing to disk.

We end up with tuning parameters:
more frequent merging blocks writes and reduces some throughout, but mitigates how many overlays a single read must check;
less frequent merging allows for a higher write throughput, but at the expense of less read throughout

There are also targeted optimizations you can add to mitigate problems.
For example, bloom filters to fast reject whole overlays
Also you can use different data structures for in memory vs on disk overlays

So that’s overlay/merge

LSM was originally published in the form of a b-tree and was thus named the log structured merge tree. If you subscribe to overlay/merge terminology, this basically just means we’re doing overlay/merge on a very large b-tree
You could just as well have a log structured merge DAG

Originally this was all formulated as a neat toy for databases and file systems.
You can take a randomized read/write pattern for a tree and do a fairly good job converting that to sequentialized writes only.

Then this got into distributed systems.
In parallel with the introduction of LSM, distributed systems researched were working their way up to creating relatively efficient append-only replicated file systems
Then someone clever realized once they had an append only file system, they can slap overlay/merge on top to make that essentially a distributed database

That idea has been implemented multiple times.
See google bigtable
See azure storage
I’m sure there are more

And that’s all she wrote!

---

A useful metaphor for what overlay/merge is might be what TCP does for IP. Namely, we take a weak primitive and on top of that build a “stronger” primitive with some overhead.

With TCP/IP, that’s reliability. IP is unreliable, but TCP is “usually reliable.” The way it does this, however, introduces some overhead, but is still useful because we can’t make IP reliable (the internet will probably never be circuit-swotched)

With overlay/merge it’s random access. Underneath overlay/merge is an append-only/immutable data structure, but overlay/merge presents a random access/mutable data structure on top. To achieve this, however, involves overhead: namely, the merge step, and read amplification for incomplete merges.

In both cases, we had a physical world problem that prevented us from building the strong primitive we wanted. For the internet, it’s the cost-infeasibility of circuit switching, and for storage it’s the physical differences between sequential and randomized I/O at the physical layer. These limitations forced us to implement a weaker primitive, but we can approximate the stronger primitive in most cases. That is what both TCP and OL/M do.

## Older brain dump

We can probably design this thing for first principles. Something like the following.

We have some kind of journal or log to which we only append.

We also want to build an index over the log. That is, we want a tree of pointers into the log, in order to quickly find the entry or entries we're looking for. Because reasons, this index needs to be kept up to date constantly. Also because of reasons, say the index is constantly being appended to, but is searched much less frequently.

This might seem kind of contrived, but it turned out to be much more general than the original authors foresaw. People realized that you could build a database/file system entirely around the technique of having a log and an index over that log. Then people realized if you did this, you could make this into a distributed database with easy-to-write performant code for data replication. Now this technique underlies many real distributed database systems, including BigTable, Cassandra, Azure Storage, to do others?

Until we successfully build the data structure we wanted, we will assume the system never crashes. We'll come back to crashes and crash recovery at the end of the article.

Building this index is pretty easy to start out. Just allocate some heap space and build a binary tree. Or your favorite other type of tree.

Eventually we'll run into problems scaling this. Say the log now grows so large the index cannot fit only in RAM. Now things get a lot trickier all of a sudden.

Writing to disk is very slow compared to writing to memory. We said the index needs to always be up to date, but if we have to go update the index on disk every time we get a write command, write throughout will greatly suffer.

A solution to this problem: amortize the disk writes. That is, batch up a bunch of new log writes in memory, and write them all out to disk in one go.

How many writes to we cache before writing out to disk? That's some basic math. Look at the rate writes are coming into memory, and look at the disks write speed. The break even point gives the number of writes to batch

The next problem: if we need to write parts of our tree to disk, how should we structure the disk section of the tree? The disk being slow is also makes this tricky.

An obvious answer from the study of file systems is the b-tree. See Wikipedia for a great overview of b-trees and how they work.

The nice thing about a b-tree is they require few disk operations per insert. If your node can accommodate 20 children, the number of node updates needed to insert an item in the worst case is log_20(the current size of the tree).

However, if we implement this and profile it, we'll soon discover another big problem: although you don't need to update many nodes when updating a b-tree, updating those nodes requires random seeks on disk, and seeks are incredibly slow (get some real numbers to back this up)

So the next question becomes, can we further reduce how often we need to seek on disk when we flush the memory tree out to the disk tree?

Observation: in any b-tree that is relatively full, the vast majority of the nodes in the tree are at the leaves. (Motivate the math which makes this converge to (N-1)/N, then for real numbers like breadth=10, 90% of nodes are at the leaves, and for breadth=100, 99% are at the leaves)

So it stands to reason we don't need to store the internal nodes of the b-tree on disk; there won't be too many of them at all. Instead we can keep the internal nodes in memory, and the leaves on disk. That should remove the burden of updating the b-tree on disk, as now the only thing we must write to disk is the leaf, which is just one write.

To review, now we have two trees in memory: a "new data" tree based on some write performance-optimized data structure, containing new data that is not yet on disk, and a "disk index" tree that's space-optimized and points to disk sectors. The disk itself just contains arrays of leaf nodes that write out in batches.

That's pretty much optimal in terms of removing disk seeks during writes. However, we still need to worry about disk seeks during reads. If we're not careful about how we write out leaf arrays to disk, we can end up with highly fragmented streams that are hard to search.

To reduce read seeks, want to keep the on disk lead array sorted. Since the "new data" memory tree is also sorted, an obvious way to do this is to build a new leaf array using the "merge step" of the merge sort algorithm (that is, zip the two sorted subarrays together into one sorted final array). After zipping, write the new leaf array to a new contiguous region in disk, without overwriting anything, and then update the "disk index" tree in memory. Then we can drop the old leaf array on disk.

We can't afford to rewrite the whole tree every time we flush changes from memory to disk; but we can do so in batches on every flush. Start at the left end of the tree and merge N disk nodes with N memory nodes to create a new 2N-node extent. Next time, merge the next N disk nodes with the next N memory nodes to create another 2N-node extent. Eventually, when you complete the tree, go back to the beginning and start again. The LSM paper calls this a rolling merge.

Motivate this reduces read seeks and write seeks for the index.

A nice side effect of this is it provides a natural way to garbage collect stale nodes and remove deleted data. When we complete a full rolling merge, all old extents are no longer in use, and we can reuse that space for new extents. Any deleted nodes are simply not added to new extents during the merge steps, so after a complete rolling merge all deleted nodes are gone.

Then introduce crashing and recovery, still need to read about this.

Finally, bring it back to distributed systems.

