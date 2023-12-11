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

If designing an algorithm that checks all these boxes sounds easy, believe me, it's not! But if it sounds daunting, rest assured it is indeed possible. It took a lot of smart people a very long time to find a solution, but they did find one in the end. This problem is hard, but solvable.

## A Programming Interface

Given what we now know aobut requirements, let's think about what kind of API we give applications who want to use the consensus algorithm we're about to invent. Maybe it'll tell us something important about how to structure our implementation.

We said before that consensus algorithms collect proposals, choose one, and replicate it to everyone in the network. So we probably want to split our interface into two methods:

* **propose()**: offers the consensus system a value you'd like to change the state to
* **query()**: ask the consensus system what value the state actually was changed to

Code that wants to update the shared state first calls **propose** with the desired final state, followed by **query** to see what value was chosen. Code that just wants to read the shared state can skip **propose** and just call **query**. The consensus system guarantees every **query** call returns the same value, as required by the coherence and no-decoherence rules.

Let's see how this would work in practice. Consider the lock server example from before:

We can make this work by defining a shared variable called `owner`, which is defined as the ID of the node which currently holds the lock, or `null` if no node currently holds it. Initially, `owner` is null. When a node wants to acquire the lock, it proposes its own ID as the value of `owner`, and then calls query to see whether it got the lock, like this:

```
// returns true if this node got the lock, false otherwise
try acquire lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

If many nodes call this simultaneously, only one node's proposal will be selected, and every node's **query** call will return that node's ID. The node with that ID will then return true, and do whatever it needs to do under the lock; all other nodes will see they don't have the lock, and do someting else.

## First Stab at a Design

To, recap, we want to build a consensus algorithm that runs on a network of computers, keeping some kind of shared state in sync. It provides two methods that can be called on any of those computers, any number of times concurrently:

* **propose**: accepts a value the caller would like to write to the shared state
* **query**: returns the current value of the shared state to the caller

We want these methods to provide the following guarantees:

* **coherence**: all computers see the same return value from query()
* **conflict-resolution**: if there are multiple propose() calls, one of them is chosen arbitrarily to be the value the system coheres on
* **no-decoherence**: if query() returns a particular value, you can safely assume all past and future calls to query() have/will return that value on every node
* **fault-tolerance**: the algorithm works even if computers, networks and software fail

To keep the problem scoped for now, we'll add a couple of simplifying assumptions:

First, we'll initially assume we're trying to reach consensus on a single scalar variable, such as an int, bool, string, enum, etc. We won't tackle lists, maps, or other complex data structures for now. But we can probably figure out how to do that if the need ever arises.

Second, we'll build a one-shot consensus primitive: once a proposed value has been chosen, it'll be impossible to ever change it again. This is probably too simple for real-world use cases; it would mean that a lock, once acquired, cannot be released, or a key-value pair, once written, cannot be overwritten. But once we have a one-shot consensus primitive, there are probably clever ways to upgrade it into a consensus primitive that supports overwrites

These might be somewhat unrealistic design limitations, but as it turns out, it'll be plenty hard to build one-shot consensus over a single variable anyways! That makes is a good place to start.

## A Leader-Based Algorithm

To get us started, let me propose a basic design:

Given a network of computers, we pick one of the nodes in the network and designate it the **leader**. Maybe a human sets a config option on all nodes in the network so they know which node is the leader. We then make the leader the center of all activity: it receives all proposals, decides which one to accept, and fields requests to get the currently accepted proposal. 

The network looks kind of like this:

[diagram]

The client API in this system is simple; it just forwards all method calls to the leader using network requests:

```
consensus {
  leader: Node; // configured by human
  
  propose(proposal) {
    leader.send_proposal(proposal)
  }
  
  query() {
    return leader.request_current_value()
  }
}
```

The leader isn't very complex either. It just remembers the first proposal received by any client, and returns that proposal upon client request:

```
leader {
  value: Proposal; // the accepted proposal
  
  init {
    value := null // no accepted proposal yet
  }
  
  on client proposal {
    // accept the first proposal, ignore others
    if (value == null) {
      value := proposal
    }
  }
  
  on client query {
    // return the current value
    client.respond(value)
  }
}
```

In short, the whole network is reading and writing a single variable stored in the leader node's local memory. The leader can access that variable directly; all other nodes read and write the variable indirectly by sending the leader network requests.

Is this a valid consensus algorithm? Let's check:

* **coherence**: ✅ &mdash; once the leader's `value` has been set, every query method call returns that value.
* **conflict resolution**: ✅ &mdash; if multiple clients make proposals, the leader only picks one of them (namely, the first one it received)
* **no-decoherence**: ✅ &mdash; the leader never changes the accepted proposal once it has been initialized the first time
* **fault-tolerance**: hmm ... we might have a problem here.

At least the way we've designed the algorithm so far, it would seem the leader is a single point of failure &mdash; if it crashes, or loses power, or gets disconnected from the network, or any number of other bad things happen to the leader, nobody else is going to be able propose or query the consensus variable. That's no good.

But maybe we can rescue this design? What if we had the leader make backups on other nodes, and promoted one of those backup nodes to become the new leader if the original leader goes offline? It's a cool idea, but it turns out not to work for one simple reason:

## Appointing Leaders is a Consensus Problem

You see, a leader-based consensus algorithm relies on every node agreeing which node is currently the leader. If different nodes obey different leaders, Very Bad Things (TM) can happen.

Consider the following network. Node 1 is currently the leader; nodes 2-5 are following. Nobody has made a proposal yet, so the current consensus `value` on the leader is `null`:

[diagram]

Now let's say the network faults, splitting the network: now nodes 1-2 can talk to each other, but not to ndoes 3-5; similarly, nodes 3-5 can talk to each other, but not nodes 1-2. We started with two sub-networks:

[diagram]

This situation, where groups of nodes can talk to other nodes within a group but not the rest of the network, is called a **network partition**. It can happen in practice, for example, if a network switch which connects the two groups of nodes loses power or something.

Well, here's the problem: node 1 was the leader, and nodes 3-5 cannot talk to it; so they're going to want to fail over to a new leader. Let's say they pick node 3 as the leader. Uh oh, now there are two leaders!

[daigram]

A situation where an algorithm can end up with more than one leader when it expects only one is called **split-brain**. Here, split-brain can result in different leaders deciding on different proposals; for example, if node 2 proposes <span style="color:blue">blue</span> and node 4 proposes <span style="color:red">red</span>, then the different sub-networks end up with different accepted proposals:

[diagram]

Now a client that queries the system can either receive <span style="color:blue">blue</span> or <span style="color:red">red</span>; clearly the coherence property is not upheld.

What we *really* need is for every node in the system to agree which node is the leader during a failover. A usable algorithm for appointing a leader during a failover situation would need to provide:

* Coherence: every node agrees which node is the leader
* Conflict resolution: it's not an error for different nodes to disagree who should be leader
* No-decoherence: nobody even temporarily believes the wrong node is the leader
* Fault-tolerance: leaders can be appointed even if there are hardware or software faults

Yup, that's right: appointing a leader *is* a consensus problem. So if we don't know how to make a consensus algorithm yet, we don't know how to make a working leader appointment algorithm yet either. We'd best avoid depending on leader nodes for now.

So where does that leave us? The starting example I chose didn't work out, but we did learn something important in the process: we need to design a **peer-to-peer** algorithm. Instead of putting one node in charge, we need ot set things up so nodes cooperate as equals, haggling out which proposal should be accepted.

## Second Stab at a Design

Do you already know any peer-to-peer consensus algorithms? I'll bet you use on in real life:

Say you're with a group of friends, and you all decide you want to eat at a restaurant for lunch, but you don't already know what restaurant you all want to go to. What do you do? Well, maybe someone throws out a restaurant, someone else throws out a different one, someone agrees, someone disagrees. Before you know it, a group opinion has formed, and once it's clear most of you want to go to that curry pizza place down the road, everyone else falls in line. Boom, consensus!

Kinda sounds like voting ... do you think we could make an algorithm out of that?

## The Majority-Rules Voting Algorithm

We can do something pretty similar to the example above in code, with one big caveat: in the real-life friends example, everyone had their own opinion about what restaurant they wanted to go to; in our algorithm, we're resolving conflicts completely arbitrarily. So we can do a similar kind of voting thing, except that every node will 'vote' for literally the first proposal it hears about.

Oh, and one more problem we'll ignore for now and fix later: TODO only two values, otherwise you can end up with no majority that would be bad.

With that, here's a basic plan:

Every node in the network will have its own local accepted proposal variable, which will keep calling `value`. Initially, `value` is null. When someone calls **propose()**, we set the local `value` to that proposal, "voting" for it. We then send the proposal to all other nodes; when the other nodes receive our proposal, they also "vote" for it by setting their local `value` variables if they haven't already voted for something else yet.

A node can only vote for one value, and can never change its vote once set &mdash; so if a node votes for someone else's value, and then the caller calls **propose()** locally, nothing should happen, because this node already voted for something else and cannot change its vote.

Finally, we implement **query()** by tallying votes: see which proposal we voted for, as well as what proposal all our peers voted for. If one proposal has been accepted by more than half of the nodes, we declare it the winner, and return that value. Boom, consensus!

Here's the same, this time as pseudocode:

```
consensus {
  value: Proposal; // the accepted proposal
  
  init {
    value := null // no accepted proposal yet
  }
  
  // this is the propose() API
  propose(proposal) {
    if (value == null) {
      value := proposal
      peers.all.send(proposal)
    }
  }
  
  // received a proposal from a peer
  on peer proposal(proposal) {
    if (value == null) {
      value := proposal
    }
  }
  
  // the query() API
  query() {
    counts: map<proposal, int>
    counts.add(this.value)
    foreach peer {
      counts.add(peer.request_value())
    }
    
    return get_majority_proposal(counts) 
  }
  
  on value requested(peer) {
    peer.reply(value)
  }
}
```

So, does *this* work?

* **coherence**: ✅ &mdash; if every node votes for only one proposal, only one proposal can reach a majority, and that's the proposal **query()** always returns
* **conflict resolution**: ✅ &mdash; the voting system allows for multiple proposals and decides in a basically arbitrary way which will be accepted
* **no-decoherence**: ✅ &mdash; nodes cannot change their votes, so once a proposal reaches majority, it cannot lose its majority
* **fault-tolerance**: . . . I told you this would be tricky, didn't I? 

No, this algorithm isn't fault-tolerant either! But at least this time around, the failure mode that breaks the algorithm isn't quite as obvious!

















