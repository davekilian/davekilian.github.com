---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

Distributed consensus algorithms are a critical piece of modern computing infrastructure that almost nobody really understands. The very first one, *Paxos*, was initially met with met with indifference; people thought it was some kind of elaborate joke. (It probably did not help that Leslie Lamport decided to dress up as Indiana Jones while presenting his algorithm to the world.) For the next 25 years, there was really only one vaible algorithm for distributed consensus, and that algorithm was Paxos. For systems built in the past decade, there has also been a second choice: *Raft*. Raft exists because Paxos is too difficult; in fact, the Raft paper is called *In Search of an Understandable Consensus Algorithm*. However, Raft is also fundamentally a lot like Paxos; it is better factored and the mechanics are a little easier to explain, but the fundamental approach to solving consensus is basically the same (and just as weird-looking) as Paxos.

Not everyone thinks Paxos is impossible to understand. Its inventor for one claims that Paxos is "among the simplest and most obvious of distributed algorithms," and that it "follows unavoidably from the properties we want it to satisfy." Although it's fun to joke this is because Dr. Lamport exists in some astral plane that we mortals can only hope to one day glimpse, the fact remains that he's right: the confusing part about Paxos isn't Paxos, it's understanding the problem space and the solution space well enough to see why you end up needing to solve consensus exactly the way Paxos does it.

In this post, we're going to try it that way. First we're going to discuss the problem of consensus, to see why it matters and also understand the mechanics of what a consensus algorithm needs to do. Then we're going to try out several "obvious" algorithms that see that they don't work. In trying to figure out why they don't work, we'll end up discussing the *FLP result*, which uses the language of formal proves to explain what is wrong with the obvious approaches we tried so far, and eliminates entire classes of things that seem like they should work but don't. When we redouble our efforts using what the FLP result tells us, we'll stumble right into Paxos &mdash; or, more specifically, the core "synod" algorithm that underlies the algorithm.

# Part 1: Consensus

One of the fun (or maybe "fun") parts of programming distributed systems is that trivial things you normally do in normal code turn out to be impossible, or boderline impossible in a distributed system. One is a problem I will call **exactly once**: how do you write code to make something happen exactly one time?

## Example: Exactly-Once

For example, say we have a function called `thingy()` and we want to make sure the program calls `thingy()` exactly one time. This is incredibly simple to do in regular, single-threaded code: you just call `thingy()`:

```java
thingy();
```

In multithreaded code, it might be a bit trickier: maybe we have a bunch of threads that all *can* call `thingy()`, but we want to make sure only one thread actually does it. We could accomplish that by putting `thingy()` behind a lock:

```java
synchronized (thingyLock) {
  if (!thingyAlreadyCalled) {
    thingy();
    thingyAlreadyCalled = true;
  }
}
```

The boilerplate is unfortunate, but otherwise the solution is simple.

What about in a distrubted system? If we had some kind of 'distributed lock' primitive, maybe we could reuse the solution above, but for now let's assume we have no fancy abstractions or tools, just the ability to send and receive messages on the network. Then how do we ensure only one machine calls `thingy()`?

Here's one idea: for most distributed systems running in data centers or in the cloud, we know ahead of time the full set of nodes that are going to participate in the distributed system. We can provision each node with a list of all nodes in the system, and then run an algorithm over that list to choose which node is the one that gets to call `thingy()`. One simple way to do this is the so-called *bully algorithm*, which works on the principle "the biggest guy wins:" pick some sort attribute of each node (maybe an integer "node ID") and then choose the node with either the largest or smallest value in that sort order. Example:

```java
int bullyAlgorithm(Collection<int> nodeIds) {
  return Collections.max(nodeIds); // min() would work too!
}
```

This algorithm does not require nodes to send or receive any network messages to decide which node gets to be the one to call `thingy()`; instead, we give each node the same node list, and run the same determinstic algorithm over that list, which causes each node to independently come to the same conclusion as to which node should be the `thingy()` caller. 

With this, we can now solve the exactly-once problem for distributed systems, even ones which use multithreading on each node:

````java
boolean iAmTheLeader = bullyAlgorithm(nodeList);
if (iAmTheLeader) {
  synchronized (thingyLock) {
    if (!thingyAlreadyCalled) {
      thingy();
      thingyAlreadyCalled = true;
    }
  }
}
````

The code has gotten more complex once again, and provisioning that node list is a little bit of an ops burden, but overall this seems pretty tractable!

## Another Example: Happened-or-Not?







TODO then do **happened-or-not**. The example problem should be trying to cancel an order that is still being placed. Was the cancellation successful?

