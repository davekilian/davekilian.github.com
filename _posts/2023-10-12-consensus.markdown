---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos.

Consensis a problem you'll run into sooner or later if you work on any kind of distributed service; every distributed system has a consensus algorithm running somewhere inside. Having a grip on consensus algorithms is incredibly useful, even if you never write one yourself!

This is a self-contained guide to how consensus algorithms tick, without the usual jargon and handwaving. The only background you'll need is enough experience to pass an undergrad data structures and algorithms course, and a basic working understanding of computer networks. We don't have the space to describe a full, production-ready system, but we will arrive at a working implementation of the core Paxos algorithm &mdash; which is the basis for everything else.

## What is Consensus?

A consensus algorithm is a protocol for keeping a network of computers in sync.

Keeping computers in sync on a network is tricky. Networks aren't reliable: transmissions can be lost before reaching their destination. Computer hardware fails; software crashes. Although your computer and the networks you rely on day-to-day are probably reliable enough that you don't have to worry about this very often, I'm sure you've run into problems like these from time to time.

Think what would happen if you were managing not one computer, but 1,000, or 10,000, or a million of them, all connected in one big network. What are the odds *none* of those computers are having a problem right now? Very low! In fact, at some point you stop asking *whether* you have any failing cmponents, to *how many* are failing right now. If you want your system to work, then, the software has to work even when some parts of the network aren't.

That's what consensus algorithms do! They are **fault-tolerant** algorithms for keeping a network of computers in sync, even  when some parts of the network have crashed, network transmissions are going missing, and so on.

I don't think it's overreaching to say consensus is the foundation of a distributed system. Without consensus, all you have is a big pile of computers that users can connect to; if you want users to see your service as a cohesive whole rather than a fragmented network of individual computers, you need some way to keep state in sync across the computers as users interact with your service. We call the algorithms that do that *consensus algorithms*.

There is a related class of *replication* algorithms, which also keep state in sync across a network of computers. The difference between a replication algorithm and a consensus algorithm is subtle, but important, and it's the first major topic we're going to nail down today.

## Consensus Use Cases

Let's take a pause here and think about a few examples of problems that can be solved using a consensus algorithm. This will be useful for deciding on requirements as well as for checking our solutions make sense in the bigger picture of real-world systems.

### Example 1: Key-Value Store



### Example 2: Cluster Membership



### Example 3: Locks and Assignment

 

## Properties of a Consensus Algorithm

Now that we have an idea of when we'd want to use a consensus algorithm, let's draw up some requirements: what do we need the algorithm to be able to do for us?

Unfortunately for me, the author, there isn't a well-accepted set of consensus properties I can just rattle off here. Database people have ACID (*atomic, consistent, isolated, durable*); there's no catchy acronym for consensus algorithms, so I'm just going to have to wing it here. Of course, we shouldn't let that stop us from coming up with big fancy words to name our properties! Big words are useful for sounding smart and intimidating people.

### Coherence

First, let's cover the basic correctness condition. The point of a consensus algorithm is to keep computers in sync; so at the end of the algorithm, all computers that participted should have the same state. Let's call this most basic requirement the **coherence** property.

### Conflict Resolution

What else? I think we're also going to need a **conflict resolution** property. What if two people try to update the same key of our key-value store? What if two nodes try to obtain the same lock using our distributed lock service? We need to choose one and only one correct answer. We need a way to resolve concurrent updates that conflict with one another.

However, it should be fine to resolve conflicts arbitrarily. For example, if two nodes try to grab the same lock, the consensus algorithm can grant the lock to either node; as long as either node gets the lock, we're good. We don't need to support custom conflict-resolution logic.

The need for the algorithm to resolve conflicts implies our consensus implementation needs to provide two core functions:

* **propose()**: call to offer the consensus system a value you'd like to change the state to
* **query()**: figure out what the final chosen value was

For example, to implement the lock server, you could let the shared state be the ID of the node which owns the lock &mdash; call this `owner`. Then, to try and acquire the lock, you'd **propose** `owner` be set to your own ID, and then you'd **query** `owner` to see which node actually obtained the lock. If it's you, then you have the lock; if it's some other node's ID, you don't have it!

```
// returns true if this node acquired the lock, false otherwise
try acquire distributed lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

### No Decoherence

One subtle aspect of conflict resolution is so important, I want to make it its own property:

When resolving a conflict, we need a strict no-takebacks rule! As soon as it is possible for any  **query** call to return a value, the algorithm *must* stay with that value forever. In other words, conflict resolution must involve a point of no return; the system as a whole must transition from "undecided" to "decided" in a single step and never go back. I'll call this the **no-decoherence** property.

Think of the chaos that would ensue if we didn't have this! Take a look at the try-acquire-distributed-lock snippet above; what if the consensus algorithm temporarily returns `me` for the value of `owner` but then changes its mind later? This node will think it has the lock, even though it actually doesn't! A consensus algorithm that changes its mind would not be very useful. We need to be sure that our consensus algorithm never goes back on its word.

This might seem obvious right now, but it'll be important to keep this in mind when we start thinking about how to make consensus algorithms work. The no-decoherence property implies some node needs to make a local decision that *globally* resolves the conflict for the entire network; this is the point of no return mentioned above. However, it's fine if it takes some time afterward for everyone to figure out a decision has been reached; at a given point in time, maybe some nodes know a decision has been made while others think the network is still deciding. That's fine, as long as nobody ever thinks a different decision was reached.

### Fault Tolerance

Finally, if we want our system to be highly-available, we're going to need **fault tolerance**. Sometimes computers will crash due to faulty hardware or software; sometimes they'll be unreachable due to network problems; sometimes we'll take them offline on purpose so we can upgrade software! For the system to remain working even when individual computers aren't, we need a consensus algorithm that works even when some computers are offline.

Talking about fault tolerance can be a little tricky. Usually we talk about fault tolerance by counting how many crashes a system can tolerate *in the worst case*.

For example, say we had a system can tolerate the crash of any node except for one 'special' node which, if it went offline, would cause the whole system to stop making progress. We would say this system is not fault tolerant, because it cannot tolerate even one crash in the worst case: namely, if that one crash affected the 'special' node. It doesn't matter how many non-special nodes are allowed to crash, because we consider the worst case!

### Recap

As we continue our discussion, keep these properties in the back of your mind:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: When multiple candidate states are possible, the algorithm picks one; it is not an error to have multiple candidates.
>
> **No Decoherence**: The no-takebacks rule: as soon as even one node can see a candidate was chosen, no other candidate can ever be chosen by anyone, ever.
>
> **Fault Tolerance**: The algorithm continues to work even if a some nodes go offline.

## Replication vs Consensus

With these requirements, we now know enough to say what's different between replication and consensus! The key difference is replication algorithms don't support ambiguity and cannot resolve conflicts. Replication algorithms only have support situations where it's possible to unambiguously copy state from one computer to another, and if ambiguity *is* discovered along the way, that's unexpected, so the algorithm is allowed to fail and a human will then need to intervene.

Replication makes sense when you can structure a system so that one node is a "leader" and all other node are "followers;" as long as every state change goes through the leader node, the leader node can *replicate* its state to followers unambiguously. Consensus is useful when you want to have multiple nodes changing the same state; that's when it's possible for state changes to conflict.

Replication algorithms are often simpler and faster than consensus algorithms, but there's an important catch: replication alone cannot be fault tolerant. If the leader crashes, the followers can't make progress without it. Thus, in practice, replication is often paired with consensus: an efficient replication algorithm is used while the leader is online and working; if the leader crashes, followers use a consensus algorithm to pick some other node to promote to leader. Once the new leader is chosen, replication can resume.

## A First Stab at Consensus

You probably already have a consensus algorithm or two in your day-to-day life. What do you do when you and your friends want to pick a restaurant but you can't come to an agreement? Maybe you'd start putting out your votes?

Could we create a consensus algorithm that works by having nodes vote on a final state? At a high level, it seems to fit the bill. Let's try it!

