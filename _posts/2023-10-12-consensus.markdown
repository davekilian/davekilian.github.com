---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos.

This is a discussion of how those algorithms work. It's no-holds-barred: we won't handwave or rely on tenuous metaphors. And we'll avoid jargon; what little we need, we'll define along the way. All you need is enough experience to pass an undergrad data structures and algorithms class, and have a basic understanding of computer networks. We'll end up 'discovering' the core Paxos algorithm by the end.

## What is Consensus?

A consensus algorithm is a protocol for keeping a network of computers in sync.

Sound simple? It's actually pretty tricky! Networks aren't reliable: transmissions can be lost before reaching their destinations. Computer hardware fails; software crashes. With the computer you use day to day, these problem happens rarely enough that it's not a big problem in your life. But what if you had not just one computer, but 1,000, or 10,000, or a million? What are the odds *none* of your computers are having a problem right now? At some point,  it makes sense to stop asking *whether* you're having one of these problems, and start asking *how many* you have right now. These things become a fact of life, for you and your code.

That's what makes consensus algorithms so difficult to design, yet so valuable to anyone running a distributed system: they can keep state in sync across a network of computers even in the face of these kinds of problems. In polite company, we call these kinds of problems **faults**, and we say consensus algorithms that deal with them as being **fault-tolerant**.

I don't think it's overreaching to say fault-tolerant consensus is foundational to distributed system. Without consensus, all you have is a big pile of computers that users can connect to; if you want users to see your service as a cohesive whole, you need some way to keep state in sync across the computers as users interact with your service. You need consensus!

There is a related class of *replication* algorithms, which also keep state in sync across a network of computers. The difference between a replication algorithm and a consensus algorithm is subtle, but important, and it's the first major topic we're going to nail down today.

## Use Cases

Let's take a pause here and think about a few examples of problems that can be solved using a consensus algorithm. This will be useful for deciding on requirements as well as for checking our solutions make sense in the bigger picture of real-world systems.

### Example 1: Key-Value Store



### Example 2: Cluster Membership



### Example 3: Locks and Assignment

 

## Properties of a Consensus Algorithm

Next let's think about what consensus algorithms need to do.

Unfortunately for me, the author, there isn't a well-accepted set of consensus properties I can just rattle off here. Whereas database people have ACID (*atomic, consistent, isolated, durable*), we don't have a catchy acronym or a widely accepted set of required properties. We're going to have to wing it! Of course, we should still come up with big fancy words to name our ideas; big words are useful tools for sounding super smart and intimidating people.

### Coherence

First, let's cover the basic correctness condition. The point of a consensus algorithm is to keep computers in sync; so at the end of the algorithm, all computers that participted had better end up with same state! Let's call this most basic requirement the **coherence** property.

### Conflict Resolution

What else? Another core feature we'll need is **conflict resolution**. What if two people try to update the same key of our key-value store? What if two nodes try to obtain the same lock using our distributed lock service? We need to choose one and only one correct answer. We need a way to resolve concurrent updates that conflict with one another.

At least for our purposes, it should be fine to resolve conflicts arbitrarily; as long as the algorithm decides on one of the proposed updates, it doesn't matter which one it chooses.

### No Decoherence

So far, we've decided a consensus algorithm must consider multiple proposals, pick one, and then reach a state of 'coherence' in which everyone knows what proposal was picked. We need to add another constraint on how this happens: once the consensus system enters a coherence state, it must never leave it &mdash; not in 3 years from now, nor 3 nanoseconds from now. We'll call this the **no-decoherence** property.

Think of the chaos that would ensue if we allowed for decoherence! Someone using our consensus-based lock service could see they have the lock and move on, not knowing the system was only in a temporary state of coherence. Then, if the re-enters coherence with a different node owning the lock, boom &mdash; you have two nodes that both think they hold the lock at the same time! Not a very useful lock, eh?

The no-decoherence requirement might seem obvious now, but it'll be very top-of-mind once we're designing consensus algorithms. It's an easy property to violate accidentally!

### Fault Tolerance

Finally, as we talked about earlier, we're going to need **fault tolerance** &mdash; the algorithm should work even if individual computers or parts of the network are having problems. But let's be a bit more crisp: what kinds of problems (faults) should we tolerate?

We'll use what's known as the **crash-recovery** model. When using this model, you assume each computer in the system is either fully online and working, or fully offline and doing nothing (crashed). Computers can crash at any time, but can also recover and come back online any time. Even though this model does not account for absolutely everything bad that can happen to computers in the real world, it's pretty comprehensive for being so simple: a wide variety of real-world faults really do like crashes, including power loss, network disconnects, downtime for upgrades and, of course, bona-fide software crashes.

To be fault tolerant, then, is to guarantee the algorithm keeps working even when a certain number of computers in the network have crashed and have not yet recovered. One subtle point here is we consider the very worst case when counting how many crashes an algorithm can tolerate: for example, an algorithm that has one 'special' that's not allowed to crash, but does allow any other node to crash, we'd still say that algorithm is not fault tolerant. In the worst case, even one crashed node could halt the algorithm, because the one crashed node could be the special node.

As we mentioned earlier, fault-tolerance is a hard requirement for a consensus algorithm to be practical. The fact is, a network can never be made 100% reliable, and so our software will have to deal with that.

### Recap

As we continue our discussion, keep these properties in the back of your mind:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: When there are multiple proposals, the algorithm picks one arbitrarily; it is not an error to have multiple proposals.
>
> **No Decoherence**: The no-takebacks rule: as soon as even one node can see a proposal was chosen, no other proposal can ever be seen as chosen by anyone, ever.
>
> **Fault Tolerance**: The algorithm continues to work even if a some nodes crash.

## Replication vs Consensus

With these properties, we can now say what's different between replication and consensus!

The key difference is that replication algorithms don't support conflicts. This makes them useful when state needs to be copied from one computer to another, without any ambiguity. If a replication algorithm discovers any ambiguity along the way, it's fine for it to error out and request manual intervention by a human. Consensus algorithms, on the other hand, anticipate conflits and resolve them.

Replication is common in situations where a special node, often called the 'leader,' handles all updates, and copies those updates to one or more 'follower' nodes. The leader resolves all conflits internally; followers simply see a stream of upgrades. Consensus is useful when you instead want multiple leaders changing the same state; that's how you end up with conflicts.

Although replication algorithms are less pwoerful than consensus in not being able to deal with conflicts, they're usually simpler and much faster. It's common to pair a single-leader multi-follower repliation algorithm with a consensus-based leader election algorithm to handle failovers in the event the leader crashes.

For the rest of this post, we'll worry only about consensus algorithms.

## A Programming Interface

Before we move on to the design phase, one last requirements-y thing we should do is decide what kind of programming interface a consensus system might provide to its users.

Given a consensus algorithm needs to collect proposals and resolve conflits between proposals, we probably want to split the interface into two methods:

* **propose()**: offers the consensus system a value you'd like to change the state to
* **query()**: ask the consensus system what value the state actually was changed to

Anyone who wants to update the shared state first calls **propose** with their desired final value, followed by **query** to see what actually happened. Anyone who just wants to know what the shared state is (without changing it) skips the **propose** step and just calls **query**. The consensus system guarantees every **query** call returns the same value.

We could use this API to build our lock service example by letting the shared state be the unique ID of the node which currently owns the lock; call this the `owner`. When a node wants the lock, it calls **propose** to try and set `owner` to its own ID, then **query** to see who actually got the lock. It would look roughly like this:

```
// returns true if this node got the lock, false otherwise
try acquire lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

## First Stab at a Design

Now we know what to build, so let's get right down to it. How will we make this work?

Start by considering: in real life, when you and a group of other people  need to reach an agreement but you don't already agree to begin with, what do you do? Say you're with a group of friends, and you all know you want to eat at a restaurant for lunch, but you don't know where you want to go. How do you make a group decision?

Well, it probably goes something like this: someone throws out a proposal for a restaurant; then someone else proposes a different restaurant; people start throwing out their opinions of where they'd want to go; eventually, it becomes clear that most people favor a restaurant, so that's where you all end up going. It's a lot like majority-rules voting, don't you think?

Do you think we could make a majority-rules voting algorithm to implement consensus? Let's try it out!





