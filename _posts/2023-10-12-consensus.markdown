---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos. Let's talk about how they work!

## What is Consensus?

A consensus algorithm is a protocol for keeping a network of computers in sync.

Sound simple? It's actually pretty tricky! Networks aren't reliable: messages can be lost before they get to where they're going. Computer hardware fails; software crashes. With the computer you use day to day, these problems probably don't happen enough to be a big deal in your life. But what if you were managing a 1,000 computers just like yours? Or 10,000, or a million? On any given day, what are the odds *none* of them is having a problem? At some point, you have to stop asking *whether* you're having problems, and start asking how many! Problems with the underlying platform become a fact of life, for you and your code.

Consensus algorithms keep computers in sync reliably, despite the unreliability of the infrastructure they run out. That's what makes them useful, yet so hard to design!

I don't think it's overreaching to say fault-tolerant consensus is foundational to distributed system. Without consensus, all you have is a big pile of computers that users can connect to; if you want users to see your service as a cohesive whole, you need some way to keep state in sync across the computers as users interact with your service. You need consensus!

## Use Cases

Our goal in this guide is to create a working consensus algorithm. Before we jump into design, we need to decide on requirements: what does a consensus algorithm need to do? To answer *that* question, it'd be useful to come up with some real-world examples where one might use one of these consensus thingamajigs.

### Example 1: Key-Value Store



### Example 2: Cluster Membership



### Example 3: Locks and Assignment

 

## Properties of a Consensus Algorithm

Now let's think about our examples: what do they rely on the consensus algorithm to do?

Unfortunately for us, there isn't a well-accepted set of consensus properties we can rattle off here. Whereas database people have ACID (*atomic, consistent, isolated, durable*), there's no catchy acronym for consensus algorithms. We're going to have to wing it! Of course, we should still come up with big fancy words to name our ideas; big words are useful tools for sounding super smart and intimidating people into thinking we must be right.

### Coherence

First, let's cover the basic correctness condition. The point of a consensus algorithm is to keep computers in sync; so at the end of the algorithm, all computers that participted had better end up with same state! Let's call this most basic requirement the **coherence** property.

### Conflict Resolution

What else? Another core feature we'll need is **conflict resolution**. What if two people try to update the same key of our key-value store? What if two nodes try to obtain the same lock using our distributed lock service? We need to choose one and only one correct answer. We need a way to resolve concurrent updates that conflict with one another.

At least for our purposes, it should be fine to resolve conflicts arbitrarily; as long as the algorithm decides on one of the proposed updates, it doesn't matter which one it chooses.

### No Decoherence

Another observation: resolving conflicts alone isn't enough. We need to add a constraint: the algorithm needs to make a decision once and for all, and never go back on it.

People are going to use consensus to make decisions, e.g. to decide whether or not a key-value overwrite was successful, a node is in a cluster, a thread obtained a lock, etc. A consensus algorithm that can report one decision, but then change it's mind later, would be useless! You wouldn't be able to make any hard decisions. For example, if consensus says you got a lock, but it can change its mind and take away your lock at any time, what good is your lock?

We need to ensure once a decision is (or can be) communicated to a client, the decision is final; it cannot be changed 3 years from now or 3 nanoseconds from now. Let's call this no-takebacks rule **no-decoherence**.

### Fault Tolerance

And finally, we know from earlier that problems like spotty networks, hardware failures and software crashes are a fact of life. These kinds of problems are called **faults**, and we our consensus algorithm should work despite them; it should be **fault-tolerant**.

A good consensus algorithm should work despite hardware failures, software crashes, power loss, machines being taken down for upgrade, network disconnets, slow networks, and lost network messages. This should apply to every computer, every network connection, and every network message equally: e.g. it should be ok for *any* node to crash, or *any* network message to go missing, at any time.

### Recap

As we continue our discussion, keep these properties in the back of your mind:

> **Coherence**: Every computer ends up with the same state.
>
> **Conflict Resolution**: When there are multiple proposed states, the algorithm picks one arbitrarily; it is not an error to have multiple proposals.
>
> **No Decoherence**: The no-takebacks rule: as soon as even one node can see a proposal was chosen, no other proposal can ever be seen as chosen by anyone, ever.
>
> **Fault Tolerance**: The algorithm continues to work even if a some nodes crash.

If designing an algorithm that checks all these boxes sounds easy, believe me, it's not! But if it sounds daunting, rest assured &mdash; it took a lot of smart people a very long time to find a solution, but they did fine one. This problem *can* be solved.

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

## The Majority-Rules Voting Algorithm



