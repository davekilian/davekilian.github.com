---
layout: post
title: Zero to Consensus
author: Dave
draft: true
---

<div markdown="1" style="width: fit-content; border: 1px solid #eee; padding-left:1em; padding-right: 1em">

**Table of Contents**

1. [Consensus](#part1)
2. [FLP](#part2)
3. [Paxos](#part3)

</div>

Ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" and they'll probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos. 

I find these algorithms kind of amazing. For one, they’re the bedrock on which the entire online world is built. If a system is distributed and manages any kind of shared state, without fail, there’s a consensus algorithm running in there somewhere. The average dev doesn’t usually need to interact with consensus algorithms directly, but the databases and cloud services we build the online world on are themselves built on consensus algorithms. Consensus is everywhere!

It’s kind of incredible that these algorithms exist at all. They do things that feel like it should be impossible to do. The environment of distributed systems is unforgiving; platforms provide very few guarantees, and the guarantees you get on paper don’t always hold in practice due to hardware malfunctions, software crashes, networks glitches and so on. It’s a world where anything that can go wrong will go wrong, is going wrong, and has been going wrong for weeks unnoticed. It’s like a perverse game of Simon Says, where you take dependencies on what you think are already very weak assumptions only to find that &ndash; Simon didn’t say! &ndash; that assumption can break too. Look at the world that way, and it seems almost impossible that an algorithm running in that environment could ever provide complete and utter certainty, reliably; yet that’s exactly what consensus algorithms manage to do! No wonder they’re everywhere.

On top of all that, it’s a small miracle that we managed to discover a working consensus algorithm at all. The first one we discovered was the culmination and many years of work by many very intelligent people, and that process was full of false starts and wrong directions. Throughout the process, people were all but certain an impossibility proof was just around the corner. In fact, even when a working algorithm was first published, people didn’t get it &mdash; people came out of the author’s presentation thinking it was a big elaborate joke. (That the algorithm was presented by a guy wearing an Indiana Jones getup probably didn’t help.)

Today, the struggle to develop and understand consensus algorithms continues. One of the biggest recent advancements in this space was published in a paper titled *In Search of an Understandable Consensus Algorithm*, and that’s 25 years after the first working algorithm was published!

---









TODO I find them kind of amazing, both in what they can do and in what a valiant struggle it was to come up with a working one. And the struggle isn’t over - 



--

These little-understood algorithms power every cloud service you rely on. Without them, you wouldn’t have the cloud; you’d just have a pile of servers you’d have to manage individually. Anytime you have a service that runs on multiple servers, where a

TODO borderline amazing they exist at all. In a world of anything that can go wrong will go wrong, is going wrong, has been going wrong for months; consensus algorithms provide a level of certainty that we have no business 

TODO but designing one was no mean feat. The title of the Paxos paper. Many failed attempts to explain these. Repeat their folly.

TODO outline-

---

If you've heard of these algorithms, you're probably also aware of their reputation. When you still have people naming landmark papers things like *In Search of an Understandable Consensus Algorithm* 25 years after the first working consensus algorithm was published, you know something's up! Many people have tried and failed to write understandable explanations of Paxos. In this guide, I will repeat their folly.

We're going to retrace the line of thinking that led to the original Paxos algorithm. This guide will be long, because it's self-contained and complete: if you can pass an undergrad programming class and write an app that sends requests and responses on a network, you have enough background to get through this thing. I won't use big words or mathematical notation when I don't have to, but I'm not going to go easy on you either &mdash; no handwaving, silly metaphors or dumbing things down. At the end, you're going to understand the core Paxos algorithm, and you'll understand the train of thought that led us there.

In part 1, we'll start with by exploring the problem space. We'll nail down exactly what a consensus algorithm does, and we'll try to design one ourselves &mdash; only to find that it's not such an easy thing to do! In part 2, we'll talk about FLP, which tells us why the things we were doing in part 1 didn't work. Finally, in part 3, we'll build on what we learned to fix our broken designs and end up with Paxos &mdash; the first working consensus algorithm.

Should be a fun ride! Buckle in!

<center>
  <a name="part1"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 1: Consensus
  </h1>
</center>

## What is Consensus?

A consensus algorithm is a protocol for keeping a network of computers in sync. They also resolve conflicts between computers in a network: if two different computers try to make different updates to the same shared state at the same time, only one of the updates will succeed, and both computers will find out once and for all which update was accepted. Just like in real life, coming to consensus in a computer network involves disputes being resolved, and the final answer being known to all.

## Why Does Consensus Matter?

Think of a service like GitHub. GitHub stores a lot of code; a lot of people push and pull from Git repositories hosted on GitHub every day. It's too much data and too much load for any one computer to handle; so we can be pretty sure GitHub is a network of computers working together. But do you have any idea how GitHub's network is laid out? I don't &mdash; I just treat GitHub like one really big computer, and leave it to the GitHub peeps figure out how to stitch everything together so it looks like one cohesive service, not a disparate pile of computers on a network. Consensus algorithms are key to doing that.

A consensus algorithm allows a distributed service to ensure that some update happened, once and for all, and cannot be overwritten or forgotten. This makes them foundational to the field of distributed systems; without them, you can't have one cohesive service, because not all servers agree on the current state of users' data.

## Why is Consensus Hard?

Let's talk about reliability for a second. Think about the device you're using to read this guide; have you ever had weird little problems with it? Frozen or crashing apps, weird glitches, system-wide slowdowns, overheating, weird network disconnects, blue screens, and so on? Most likely these kinds of things have happened to you, even if they don't happen enough to disrupt using your device day to day.

But now imagine you were using not just one device, but a thousand, or even a million of them across the world, connected by thousands of miles of network cables. How often do you think you'd be dealing with these kinds of little problems? Heck, you'd probably never be free of them, no matter how hard you tried! Rare problems, multiplied by thousands of machines, or millions of requests per minute, become common. Somewhere in your system, you will have a machines overheating, crashing, getting disconnected from the network, and so on. You can't fix these problems and make them stay fixed; so, your code has to deal with the contingincies of computers having these problems.

Consensus algorithms that can be deployed in practical, real-world settings must be designed to work despite hardware failures and software glitches. In technical terms, we say consensus algorithms are **fault-tolerant**, tolerant of "faults" in the underlying hardware or software.

## Use Cases

So far we have a pretty vague idea of what consensus is or what it does for us. Let's get more concrete. Here are some situations people commonly deploy consensus algorithms:

### Example 1: Key-Value Store



### Example 2: Cluster Membership



### Example 3: Locks and Assignment

 

## Properties of a Consensus Algorithm

Think about our examples above. What do they all rely on the consensus algorithm to do?

Unfortunately, there isn't a well-accepted set of consensus properties we can rattle off here. Database people have ACID (*atomic, consistent, isolated, durable*); there's no similarly catchy acronym for consensus algorithms. We're going to have to wing it. Of course, we still ought to come up with big fancy words to name our ideas; gotta sound smart if we want to be taken seriously! (Also, having 1-2 word names for these will be useful, since we're going to refer back to these quite a bit.)

### Coherence

First, let's cover the basic correctness condition. The point of a consensus algorithm is to take some state, and keep it in sync across a network of computers; so when a consensus algorithm finishes, all the computers had better end up with the same state! Let's call this most basic requirement the **coherence** property.

### Conflict Resolution

What else? Another core feature we'll need is **conflict resolution**. What if two people try to update the same key of our key-value store at the same time? What if two nodes try to obtain the same lock using our distributed lock service in parallel? When multiple answers are possible, we need to choose one and only one correct answer. We need a way to resolve concurrent updates that conflict with one another.

At least for our purposes, it should be fine for the algorithm to resolve the conflicts however it chooses, even at random if it wants to. The algorithm does not ever need to prefer one kind of answer over another; it can resolve conflicts arbitrarily.

### No Decoherence

Another observation: resolving conflicts alone isn't enough. We need to add a constraint: the algorithm needs to make a decision once and for all, and never go back on it. As soon as the algorithm tells any computers, "this is the answer that everyone has agreed upon," it must be impossible for any other computer to ever think that we agreed on some other value.











---

An alternate way to explain the same idea:

* During the algorithm, there must not be a time where anyone can see a wrong answer
* Equivalently: once the conflict is resolved, we must never change our minds
* Think of the chaos that would ensure without that guarantee!
* But note this rule is actually quite strict. We want it to apply for miniscule timescales
* The *instant* someone can tell the conflict is resolved, we can never change our minds

---





Another observation: resolving conflicts alone isn't enough. We need to add a constraint: the algorithm needs to make a decision once and for all, and never go back on it.

People are going to use consensus to make decisions, e.g. to decide whether or not a key-value overwrite was successful, a node is in a cluster, a thread obtained a lock, etc. A consensus algorithm that can report one decision, but then change it's mind later, would be useless! You wouldn't be able to make any hard decisions. For example, if consensus says you got a lock, but it can change its mind and take away your lock at any time, what good is your lock?

We need to ensure any decision that has been (or can be) communicated to a client is final: it cannot be changed 3 years from now or 3 nanoseconds from now. Let's call this no-takebacks rule **no-decoherence**.

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

If designing an algorithm that checks all these boxes sounds easy, believe me, it's not! But if it sounds daunting, rest assured it is indeed possible. It took a lot of intelligent people a long time to find a solution, but they did find one in the end. This problem is hard, but solvable!

## A Programming Interface

Given what we now know about requirements, let's think about what kind of API we want give to clients of our consensus algorithm. Maybe it'll tell us something important about how to structure our implementation.

If we want the conflict-resolution property to really make sense, we need to collect proposals separately from deciding on which proposal to accept. So we probably want to split our interface into two methods:

* **propose()**: offers the consensus system a value you'd like to change the state to
* **query()**: ask the consensus system what value the state actually was changed to

Code that wants to update the shared state first calls **propose** with the value it'd prefer, followed by **query** to see what value was chosen. Code that just wants to read the shared state can skip **propose** and just call **query**. The consensus system guarantees every **query** call returns the same value, as required by the coherence and no-decoherence rules.

For example, think about the lock server example from before: we could make that work by defining a shared consensus variable called `owner`, which is defined as:

* the ID of the node which currently holds the lock
* `null` if nobody has the lock yet

Initially, `owner` is null. When a node wants to acquire the lock, it proposes its own ID as the value of `owner`, and then calls query to see whether it got the lock, like this:

```
// returns true if this node got the lock, false otherwise
try acquire lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

If many nodes call this simultaneously, only one node's proposal will be selected, and every node's **query** call will return that node's ID. The node with that ID will then return true, and do whatever it needs to do under the lock; all other nodes will see they don't have the lock, and do someting else.

## First Stab at a Design

Let's get cracking! To recap, we want to build a consensus algorithm that runs on a network of computers, keeping some kind of shared state in sync. It provides two methods that can be called on any of those computers, any number of times concurrently:

* **propose**: accepts a value the caller would like to write to the shared state
* **query**: returns the current value of the shared state to the caller

We want these methods to provide the following guarantees:

* **coherence**: all computers always see the same return value from query()
* **conflict-resolution**: if there are multiple propose() calls, one of them is chosen arbitrarily to be the value query() always returns
* **no-decoherence**: if query() returns a particular value, you can safely assume all past and future calls to query() have/will return that value on every node
* **fault-tolerance**: the algorithm works even if computers, networks and software flake out on us

This is what we want to end up with, at least. For now, let's also put an additional constraint on our design: we're going to start by building a one-shot consensus algorithm. Once a proposed value has been chosen, we won't allow it to be changed ever again.

This is probably too simple for real-world use cases; it would mean that a lock, once acquired, cannot be released, or a key-value pair, once written, cannot be overwritten. But once we have a one-shot consensus primitive, there are probably clever ways to upgrade it into a consensus primitive that supports overwrites We'll focus on that later. For now, we'll worry about getting a one-shot consensus algorithm to work.

The shared variable that we're proposing values for and querying values of, can have any type: it can be an int, bool, char, string, list, map, or a tuple of any of those &mdash; as long as you're good with setting it once and never changing it, at least for now!

## A Leader-Based Algorithm

To get us started, let me propose a basic design:

Given a network of computers, we pick one of the nodes in the network and designate it the **leader**. Maybe a human sets a config option on all nodes in the network so they all know which node is the leader. We then make the leader the center of all activity: it receives all proposals, decides which one to accept, and returns the accepted proposal on queries.

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

At least the way we've designed the algorithm so far, it would seem the leader is a **single point of failure** &mdash; if it crashes, or loses power, or gets disconnected from the network, or any number of other bad things happen to the leader, nobody else is going to be able propose or query the consensus variable. That's no good.

But maybe we can rescue this design? What if we had the leader make backups on other nodes, and promoted one of those backup nodes to become the new leader if the original leader goes offline? It's a cool idea, but it turns out not to work for one simple reason:

## Appointing Leaders is a Consensus Problem

A leader-based consensus algorithm relies on every node agreeing which node is currently the leader. If different nodes obey different leaders, Very Bad Things (TM) can happen.

Consider the following network. Node 1 is currently the leader; nodes 2-5 are following. Nobody has made a proposal yet, so the current consensus `value` on the leader is `null`:

[diagram]

Now let's say the network faults, splitting the network: now nodes 1-2 can talk to each other, but not to nodes 3-5; likewise, nodes 3-5 can talk to each other, but not nodes 1-2. Our system has now split into two sub-networks:

[diagram]

This situation, where groups of nodes can talk to other nodes within a group but not the rest of the network, is called a **network partition**. It can happen in practice, for example, if a network box which connects the two groups of nodes gets unplugged or something.

Well, here's the problem: node 1 was the leader, and nodes 3-5 cannot talk to it; for all they know node 1 might have crashed. So they fail over to a new leader. Let's say they pick node 3:

[daigram]

Uh oh, now there are two leaders! A situation like this, where a system can end up with more than one leader when it expects only one, is called **split-brain**. Here, split-brain can result in different leaders deciding on different proposals: say node 2 proposes <span style="color:blue">blue</span> and node 4 proposes <span style="color:red">red</span>. Then the different sub-networks end up with different accepted proposals:

[diagram]

Now a client that queries the system can either receive <span style="color:blue">blue</span> or <span style="color:red">red</span>; clearly the coherence property is not upheld.

How do we fix this? We need every node in the system to agree which node is the leader during a failover. That way, the failover from node 1 to node 3 either happens completely or it doesn't happen at all. A usable algorithm for appointing a leader during a failover situation would need to provide:

* Coherence: every node agrees which node is the leader
* Conflict resolution: it's not an error for different nodes to disagree who should be leader
* No-decoherence: nobody even temporarily believes the wrong node is the leader
* Fault-tolerance: leaders can be appointed even if there are hardware or software faults

Yup, that's right: appointing a leader *is* a consensus problem. So if we don't know how to make a consensus algorithm yet, we don't know how to make a working leader failover algorithm yet either. We'd best avoid depending on leader nodes for now.

So where does that leave us? The starting example I chose didn't work out, but we did learn something important in the process: we need to design a **peer-to-peer** algorithm. Instead of putting one node in charge, we need ot set things up so nodes cooperate as equals, haggling out which proposal should be accepted.

Do you know of any way to do that?

## Second Stab at a Design

I'll bet you use peer-to-peer consensus algorithms all the time in real life:

Say you're with a group of friends, and you all decide you want to eat at a restaurant for lunch, but you don't already know what restaurant you all want to go to. What do you do? Well, maybe someone throws out a restaurant, someone else throws out a different one, someone agrees, someone disagrees. Before you know it, a group opinion has formed, and once it's clear most of you want to go to that curry pizza place down the road, everyone else falls in line. Boom, consensus!

Kinda sounds like voting ... do you think we could make an algorithm out of that?

## The Majority-Rules Voting Algorithm

We can do something pretty similar to the example above in code, with one big caveat: in the real-life friends example, everyone had their own opinion about what restaurant they wanted to go to; in our algorithm, we're resolving conflicts completely arbitrarily. So we can do a similar kind of voting thing, except that every node will 'vote' for literally the first proposal it hears about, instead of providing any opinion of its own.

With that, here's a basic plan:

Every node in the network will have its own local accepted proposal variable, which will keep calling `value`. Initially, `value` is null. When someone calls **propose()**, we set the local `value` to that proposal, "voting" for it. We then send the proposal to all other nodes; when the other nodes receive our proposal, they also "vote" for it by setting their local `value` variables if they haven't already voted for something else yet.

A node can only vote for one value, and can never change its vote once set &mdash; so if a node votes for someone else's value, and gets a local **propose()** method call, nothing should happen, because this node already voted for something else and cannot change its vote.

Finally, we implement **query()** by tallying votes: see which proposal we voted for, as well as what proposal all our peers voted for. If one proposal has been accepted by more than half of the nodes, we declare it the winner, and return that value. Boom, consensus!

One problem with this design already, as given, is that it doesn't support more than two proposals: with three proposals, it's possible that each proposal gets enough votes that no vote can reach majority. For now, let's just pretend this doesn't happen. But this is something we'll have to circle back to and fix later though.

Here's the algorithm again, as pseudocode:

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

So, assuming no more than two proposals so that we can guarantee a majority gets reached ... does *this* design work?

* **coherence**: ✅ &mdash; if every node votes for only one proposal, only one proposal can reach a majority, and that's the proposal **query()** always returns
* **conflict resolution**: ✅ &mdash; the voting system allows for multiple proposals and decides in a basically arbitrary way which will be accepted
* **no-decoherence**: ✅ &mdash; nodes cannot change their votes, so once a proposal reaches majority, it cannot lose its majority
* **fault-tolerance**: . . . 😀 I told you this would be tricky, didn't I? 

No, this algorithm isn't fault-tolerant either, but we're getting better at this &mdash; this time, the problem is much more subtle!

This design is pretty resilient, but depending on how things play out, it sometimes has a **window of vulnerability** where one extremely poorly timed crash could bring the entire system to a halt, preventing a majority from being reached until that computer is brought back online. This case might be rare, but it still happens as a result of just one crash, so technically we have to admit this algorithm cannot withstand just one crash &mdash; hence, it's not fault-tolerant.

Why not? Let's watch this in action:

Say we have a cluster of 7 nodes. One proposes <span style="color:blue">blue</span>, the other <span style="color:red">red</span>:

[diagram]

Off to the races! Those two nodes each send their proposals, <span style="color:blue">blue</span> and <span style="color:red">red</span>, to all their peers. Remember, each of those peers will vote for whichever proposal it receives first, then never change its mind. Since we have two proposals propagating at the same time, the choice will come down to small timing variances. Let's say the race continues for a while, and just happens to turn out neck-and-neck, three votes each for <span style="color:blue">blue</span> and <span style="color:red">red</span>. Just one node is still decided because it hasn't received a proposal yet:

[diagram]

What happens if that undecided node crashes right now?

[diagram]

Now we're in trouble. A proposal needs 4 votes to be accepted; but the voting is over, and no proposal has 4 votes! Seems like we're stuck.

Can we get the algorithm un-stuck from this point? Maybe, but the no-decoherence property makes this very tricky. For now, all we know is the state of the network is this:

[diagram with 3 red, 3 blue, one dead]

Did the node actually decide red, and tell anybody about this before it went offline?

[diagram with 3 red, 3 blue, one red in a thought bubble]

Or blue?

[diagram with 3 red, 3 blue, one blue instead of red]

Or maybe it actually hadn't made a decision yet, and it's safe to tiebreak?

[diagram with 3 red, 3 blue, one blank in a thought bubble]

If we want to move on without the node that crashed and still uphold the no-decoherence property in all cases, we really need to know whether the crashed node made a decision, and if so what it is. But the node is offline; it can't tell us what it already did. We're good and stuck now, up until we manage to get the node back up and running from where it left off.

To conclude, the loss of a single node with this algorithm is usually just fine, but in cases like the above it can be catastrophic, bringing the entire system to a halt. So, this algorithm is not fault-tolerant.

---

## Intermission

If you have the time, this would be a good point to step away from your computer and think over the above. What went wrong with our two approaches above? Can you think of a way to fix either one? Or a different strategy entirely for implementing fault-tolerant consensus?

Based on how long it took the field to come up with a working algorithm, I'm guessing you won't be able to find the solution in an afternoon walk through the woods. But thinking through a variety of approaches and seeing how they hit the same dead end will give you a better feel for the problem space, and set up for the next half of this article.

Adieu, for now! Or, for the impatient, read on . . .

---



<center>
  <a name="part2"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 2: FLP
  </h1>
</center>

Before a working consensus algorithm was discovered, people chewed through this problem just as you might have during your intermission. And they kept running into the same dead end, over and over: they could make an algorithm that provided the coherence, conflict resolution, and no-decoherence properties, and even make it *usually* fault tolerant; but there'd always be that one little case, one way things could go just a little bit wrong and then boom, one crash brings down the whole system. Every algorithm had at least one window of vulnerability, no matter what they tried.

Well, when you're trying to design an algorithm and you repeatedly run into a dead end, the next thing you should do is try to prove impossibility: that one can never design that algorithm, because that dead end will always come up no matter what you do, due to something about the problem space.

Which is exactly what three researchers did in the mid-1980s!

## When the Going Gets Hard, Prove the Going is Hard and Give Up





TODO the details are

* *Impossibility of Distributed Consensus with One Faulty Process*
* Fischer, Lynch, Paterson
* 1985





<center>
  <a name="part3"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 3: Paxos
  </h1>
</center>





