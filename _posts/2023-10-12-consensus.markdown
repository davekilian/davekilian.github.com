---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos.

If you work on, in or near a distributed system, you'll run into the problem of consensus sooner or later, whether or not you have to solve consensus yourself. Every distributed system has a consensus algorithm running somewhere inside. Having a grip on consensus algorithms is thus incredibly useful, even if you never write one yourself!

This is a small, self-contained guide to the Paxos consensus algorithm, with minimal jargon and no handwaving. The only background you'll need is undergrad data structures and algorithms. We don't have space to describe a full, production-ready system, but we can at least tackle the big ideas.

This guide will not cover Raft; I don't think Raft is a good place to start. Raft is often billed as easier to understand than Paxos, but that's comparing Raft to a full production-ready Paxos with all the bells and whistles. Paxos is nicer for learning about consensus, because it can be pared down to a tiny core algorithm with almost no moving parts.

## What is Consensus?

A consensus algorithm is a protocol for keeping a network of computers in sync.

Consensus algorithms are the foundation of a distributed system. Think about it this way: you might choose to distribute your service across a bunch of computers, maybe to scale better or provide better uptime. But you don't want users to worry about which computers they're connecting to individually; you want your service to look like it's code running on one really big computer. But since users actually are connecting to individual 'little' computers on your network, that means keeping your state in sync across all those little computers as users interact with the service. *Consensus* is our name for algorithms that do that!

There is a related class of *replication* algorithms, which also keep state in sync across computers. Nailing down the difference between replication and consensus is the first major topic we're going to tackle today. For now, the short answer is that replication algorithms just copy state; consensus algorithms deal with ambiguity and conflicting updates. We'll see what this means soon enough.

## Consensus Use Cases

Let's take a pause here and think about a few examples of problems that can be solved using a consensus algorithm. This will be useful for deciding on requirements as well as for checking our solutions make sense in the bigger picture of real-world systems.

### Example 1: Key-Value Store



### Example 2: Cluster Membership



### Example 3: Locks and Assignment

 

## Properties of a Consensus Algorithm

Now let's gather some requirements; given the kinds of things we want to use a consensus algorithm for, what does the algorithm need to be able to do?

Unfortunately for me, the author, there isn't a well-accepted set of properties a consensus algorithm needs to provide that I can just rattle off here. Database people can just say they want transaction to be *atomic, consistent, isolated, durable* and assume the reader has already been filled in on what those mean; there is no analog to ACID for consensus, we generally just kind of assume the reader already gets what the algorithm is supposed to do. So let's draw up a set of requirements on our own.

First, let's cover the basic correctness condition. The point of a consensus algorithm is to keep computers in sync; so at the end of the algorithm, all computers that participted should have the same state. Just to have a name, let's call this **coherence**.

What else?







To recap:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: The algorithm makes a decision when multiple candidate values are possible. It is not an error to have multiple proposed states.
>
> **No-Decoherence**: The no-takebacks rule: once a candidate can be seen as chosen, it is never possible for any other value to be seen as chosen.
>
> **Fault Tolerance**: The algorithm works even if a reasonable number of nodes crash.

With these requirements, we now know enough to say what's different between replication and consensus! Replication algorithms have no notion of conflict resolution; they don't need a conflict-resolution or no-decoherence property, it's fine to simply fail in the case any ambiguity is discovered. Replication algorithms make sense when you can structure a system so that one node is a "leader" and all other nodes are "followers;" then the leader can *replicate* its state unambiguously to followers. Consensus algorithms make sense when you can't have just one node be in charge, because state changes on one computer can conflict with other changes on a different computer. 

