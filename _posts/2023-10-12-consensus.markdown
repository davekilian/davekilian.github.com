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

## Consensus Use Cases

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

So far, we know the algorithm needs to resolve potential conflicts and enter a state of 'coherence' in which every machine knows what final value the system decided on. On top of this, we need to add another constraint: once the system has entered this coherence state, it must never leave it &mdash; not in 3 years from now, nor 3 nanoseconds from now. We'll call this the **no-decoherence** property.

Think of the chaos that would ensue if we didn't have this! Someone using our consensus-based lock service could see they have the lock and move on, unaware that the system then changed its mind later on and gave the lock to someone else. Then two nodes could think they both have the lock at the same time!

The no-decoherence requirement might seem obvious right now, but once we move on to designing consensus algorithms, it'll turn out to be quite tricky to guarantee this property is always held!

### Fault Tolerance

Finally, as we talked about earlier, we're going to need **fault tolerance** &mdash; the algorithm should work even if individual computers or parts of the network are having problems.











This raises an important question: what, exactly, is a fault?

---

TODO rework the below into the above. Name all sorts of faults and draw equivalence to a machine crash, where a crash just means 'the machine is not running right now.'

---

Sometimes computers will crash due to faulty hardware or software; sometimes they'll be unreachable due to network problems; sometimes we'll take them offline on purpose so we can upgrade software! For the system to remain working even when individual computers aren't, we need a consensus algorithm that works even when we have hardware and software that isn't working.

Talking about fault tolerance can be a little tricky. Usually we talk about fault tolerance by counting how many crashes a system can tolerate *in the worst case*.

For example, say we had a system can tolerate the crash of any node except for one 'special' node which, if it went offline, would cause the whole system to stop making progress. We would say this system is not fault tolerant, because it cannot tolerate even one crash in the worst case: namely, if that one crash affected the 'special' node. It doesn't matter how many non-special nodes are allowed to crash, because we consider the worst case!

### Recap

As we continue our discussion, keep these properties in the back of your mind:

> **Coherence**: Every computer ends up with the same state
>
> **Conflict Resolution**: When there are multiple proposals, the algorithm picks one arbitrarily; it is not an error to have multiple proposals.
>
> **No Decoherence**: The no-takebacks rule: as soon as even one node can see a proposal was chosen, no other proposal can ever be seen as chosen by anyone, ever.
>
> **Fault Tolerance**: The algorithm continues to work even if a some nodes go offline.

## Replication vs Consensus

With these requirements, we now know enough to say what's different between replication and consensus! The key difference is replication algorithms don't support ambiguity and cannot resolve conflicts. Replication algorithms only have support situations where it's possible to unambiguously copy state from one computer to another, and if ambiguity *is* discovered along the way, that's unexpected, so the algorithm is allowed to fail and a human will then need to intervene.

Replication makes sense when you can structure a system so that one node is a "leader" and all other node are "followers;" as long as every state change goes through the leader node, the leader node can *replicate* its state to followers unambiguously. Consensus is useful when you want to have multiple nodes changing the same state; that's when it's possible for state changes to conflict.

Replication algorithms are often simpler and faster than consensus algorithms, but there's an important catch: replication alone cannot be fault tolerant. If the leader crashes, the followers can't make progress without it. Thus, in practice, replication is often paired with consensus: an efficient replication algorithm is used while the leader is online and working; if the leader crashes, followers use a consensus algorithm to pick some other node to promote to leader. Once the new leader is chosen, replication can resume.

## A Programming Interface

That a consensus algorithm needs to support resolving suggests a way to structure the interface callers use to call our consensus algorithm. We can give callers two functions:

* **propose()**: offers the consensus system a value you'd like to change the state to
* **query()**: ask the consensus system what value the state actually was changed to

Then anyone who wants to update the shared state first calls **propose** with the desired final value, followed by **query** to see what actually happens. The consensus system guarantees every **query** call returns the same value.

For example, we could use this interface to implement the lock server by letting the shared state be the unique ID of the node which currently owns the lock; call this variable the `owner`. Then to try acquiring the lock, a node calls **propose** on `owner` with its own ID, then **query** to see who actually got the lock:

```
// returns true if this node got the lock, false otherwise
try acquire lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

## A First Stab at Consensus

You probably already have a consensus algorithm or two in your day-to-day life. What do you do when you and your friends want to pick a restaurant but you can't come to an agreement? Maybe you'd start putting out your votes?

Could we create a consensus algorithm that works by having nodes vote on a final state? At a high level, it seems to fit the bill. Let's try it!

