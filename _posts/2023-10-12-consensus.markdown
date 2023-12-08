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

Unfortunately for me, the author, there isn't a well-accepted set of properties a consensus algorithm needs to provide that I can just rattle off here; for example, whereas database people can just say they want transaction to be *atomic, consistent, isolated, durable*, in the distributed systems field there's no well-accepted set of properties for a consensus algorithm. We generally just kind of assume the reader 'gets' what the algorithm is supposed to do. So that means I, your author, will have to wing it. And wing it we shall!









To recap:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: 
>
> **No-Decoherence** (the no-takebacks rule): 
>
> **Fault Tolerance**: 



## Consensus vs Replication

