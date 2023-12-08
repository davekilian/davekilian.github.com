---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos.

If you work on, in or near a distributed system, you'll run into the problem of consensus sooner or later, whether or not you have to solve consensus yourself. Every distributed system has a consensus algorithm running somewhere inside. Having a grip on consensus algorithms is thus incredibly useful, even if you never write one yourself!

This is a small, self-contained guide to consensus algorithms, with minimal jargon and no handwaving. The only background you'll need is undergrad data structures and algorithms. We don't have space to describe a full, production-ready system, but we can at least tackle the big ideas.

We will discuss Paxos in this guide, despite the existence of Raft, a more modern alternative. Even though Raft is often billed as more understandable than Paxos, I still think Paxos is a better algorithm to learn first. A full production-ready Raft is easier to understand than a full production-ready Paxos, but Paxos can be pared down to a much smaller core than Raft.

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

First, let's cover the basic correctness condition. The point of a consensus algorithm is to keep computers in sync; so at the end of the algorithm, all computers that participted should have the same state. Just to have a name, let's call this the **coherence** property.

What else? I think we're also going to need a **conflict resolution** property too. What if two people try to update the same key of our key-value store? What if two nodes try to obtain the same lock using our distributed lock service? We need to choose one and only one correct answer. Thus we need a way to resolve concurrent updates that conflict with one another.

Luckily, we don't need to say *how* conflicts are resolved. It's perfectly fine if the consensus algorithm picks an arbitrary candidate &mdash; an arbitrary key-value store update which is accepted, or an arbitrary node to obtain the lock first. Users of the consensus algorithm will *propose* an update to be made, and the consensus algorithm will then tell the user later whether the update was accepted, or rejected in favor of a different update.

One subtle aspect of conflict resolution is so important, I want to make it its own property: it's not just enough to *eventually* resolve the conflict, we need a strict no-takebacks rule! As soon as it is possible for any node to 'see' that a candidate value has been chosen, the algorithm *must* stay with that value forever. In other words, conflict resolution must involve a point of no return; the system as a whole must transition from "undecided" to "decided" in a single step and never go back after that. I'll call this the **no-decoherence** property.

Think of the chaos that would ensue if we didn't have this! TODO

TODO of course, it's fine for other nodes to asynchronously discover that a decision has been made. The omniscient observer.

Finally, if we want our system to be highly-available, we're going to need **fault tolerance**. Sometimes computers will crash due to faulty hardware or software; sometimes they'll be unreachable due to network problems; sometimes we'll take them offline on purpose so we can upgrade software! For the system to remain working even when individual computers aren't, we need a consensus algorithm that works even when some computers are offline.

TODO when we talk about fault tolerance, we ask how many nodes can crash and consider the worst case. For example, a system with a single special node that must not crash but many non-special nodes that can crash is not considered fault-tolerant, because it cannot withstand even 1 worst-case crash; if that special node goes down we're toast.

TODO don't worry about byzantine, saddest moment

To recap:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: When multiple candidate states are possible, the algorithm picks one; it is not an error to have multiple candidates.
>
> **No-Decoherence**: The no-takebacks rule: once a candidate can be seen as chosen, it is chosen, everywhere, forever.
>
> **Fault Tolerance**: The algorithm works even if a some nodes crash.

## Replication vs Consensus

With these requirements, we now know enough to say what's different between replication and consensus! The key difference is replication algorithms don't support ambiguity and cannot resolve conflicts. Replication algorithms only have support situations where it's possible to unambiguously copy state from one computer to another, and if ambiguity *is* discovered along the way, that's unexpected, so the algorithm is allowed to fail and a human will then need to intervene.

Replication makes sense when you can structure a system so that one node is a "leader" and all other node are "followers;" as long as every state change goes through the leader node, the leader node can *replicate* its state to followers unambiguously. Consensus is useful when you want to have multiple nodes changing the same state; that's when it's possible for state changes to conflict.

Replication algorithms are often simpler and faster than consensus algorithms, but there's an important catch: replication alone cannot be fault tolerant. If the leader crashes, the followers can't make progress without it. Thus, in practice, replication is often paired with consensus: an efficient replication algorithm is used while the leader is online and working; if the leader crashes, followers use a consensus algorithm to pick some other node to promote to leader. Once the new leader is chosen, replication can resume.

## A First Stab at Consensus



