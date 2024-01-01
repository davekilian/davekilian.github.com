---
layout: post
title: Little Book of Consensus Algorithms
author: Dave
draft: true
---

<div markdown="1" style="width: fit-content; border: 1px solid #eee; padding-left:1em; padding-right: 1em">

**Table of Contents**

1. [Consensus Problems](#part1)
2. [Designing a Consensus Algorithm](#part2)
2. [FLP](#part3)
3. [Paxos](#part4)

</div>

Ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" and they'll probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos. 

These algorithms are kind of amazing. For one, they’re the bedrock on which the entire online world is built. If a system is distributed and manages any kind of shared state, there’s a consensus algorithm running in there somewhere. The average dev may not need to interact with consensus algorithms directly, but the databases and cloud services we build the online world on are themselves built on consensus algorithms. Consensus is everywhere!

It’s kind of incredible that these algorithms exist at all. They do things that feel impossible. The environment of distributed systems is unforgiving; platforms provide surprisingly few guarantees, and the guarantees you get on paper don’t always hold in practice: hardware malfunctions, software crashes, networks glitch and so on. It’s a world where anything that can go wrong will go wrong, is going wrong, and has been going wrong for weeks unnoticed. It’s like a perverse game of Simon Says, where you take dependencies on what you think are already very weak assumptions only to find that &ndash; Simon didn’t say! &ndash; that assumption can break too. Look at the world that way, and it seems almost impossible that an algorithm running in that environment could ever provide complete and utter certainty, and still work pretty darn reliably. But that’s exactly what consensus algorithms manage to do! No wonder they’re everywhere.

On top of all that, it’s a small miracle that we managed to discover a working consensus algorithm at all. The first one discovered was the culmination and many years of hard work by many intelligent people, and that process was a long journey, full of false starts and wrong directions. Along the way, people were all but certain an impossibility proof was just around the corner. In fact, even when a working algorithm was first published, people didn’t get it &mdash; people came out of the author’s presentation thinking it was some kind of practical joke! (It probably didn’t help the author was wearing an Indiana Jones getup, acting as if his algorithm was an archaeological find, but still!)

Today, the struggle to develop and understand consensus algorithms continues. One of the biggest recent advancements in this space was published in a paper titled *In Search of an Understandable Consensus Algorithm* ... and that’s 25 years after the first working algorithm was published! The original author of the first consensus algorithm published the algorithm twice, but people still don’t get it. Much ink has been put to paper trying to explain how these things work, to no avail. Many people have written blogs trying to make Paxos easy to understand, and failed; in this little book, I shall repeat their folly.

We're going to retrace the original line of thought that led to the discovery of Paxos, which is the first consensus algorithm ever discovered. Our discussion will be self-contained and complete: if you can pass an undergrad programming class and deploy your code to the cloud, you have enough background to get through this thing. I won't use big words or mathematical notation when I don't have to, but I'm not going to go easy on you either &mdash; no handwaving, stretching metaphors or oversimplifying. At the end, you're going to understand the core Paxos algorithm, and you'll understand how somebody could have come up with it.

In chapter 1, we'll start by exploring the problem space. We'll nail down exactly what a consensus algorithm does, and when you’d want to use one. In chapter 2, we’ll start trying to design an algorithm to implement that spec &mdash; only to find ourselves running into a dead end! In chapter 3 we'll talk about FLP, a major discovery that makes it clear why the things we were doing in chapter 2 didn't work. Finally, in chapter 4, we'll use what FLP taught us to fix our broken designs, and end up with Paxos &mdash; the first working consensus algorithm.

<center>
  <a name="part1"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    1: Consensus Problems
  </h1>
</center>

A lot of computing terms come from real-life metaphors. The real-life metaphor behind consensus algorithms goes something like this:

Imagine you and a group of other people are stuck on a decision. Maybe you all want to go to a restaurant but you can’t decide where to go. The conundrum is: everyone wants to go out to eat, but until you all agree *where* to go, nobody’s going anywhere; so everyone wants the same thing, but nobody’s getting what they want! Imagine some people now throw out some ideas to help get the ball rolling, not with the goal of debating the best possible place to eat, just to find an option everyone can agree to so you can move on and actually, you know, go eat. That agreement, made so the whole group can move on, is what we call *consensus*.

The dictionary definition of “consensus” is basically “agreement,” but in our context we’ll use the word to mean “agreement that lets a group move forward.” It implies making a decision that’s good enough for everyone, even if not everyone is getting exactly what they wanted; the thing everyone wants most is to be able to move on. Our definition of consensus also implies sticking to the consensus decision once you’ve moved on, even if new options are proposed or new information comes to light. Otherwise, allowing the discussion to remain open would prevent the group from moving on.

That’s a pretty good metaphor for how consensus problems work in distributed systems programming. Groups of programs can get similarly stuck on decisions. Let’s see how that happens:

## Examples of Consensus Problems

### Pushing to GitHub Repos

GitHub stores a lot of code, and a lot of people push and pull from Git repos on GitHub every day. It’s too much data and too much load for any one server to handle, so we can be pretty sure GitHub is a distributed service running across a whole bunch of servers. I don’t know about you, but I personally have never needed to worry about any of GitHub’s servers or how their network is laid out internally. I just point my browser or my git client to their URL and go manage some repos. I can do that because GitHub presents itself as a single cohesive service, abstracting away from me the details of what servers they run code on or how their network is laid out. Services like these are rife with potential for conflicts. 

Say we have two users, Alice and Bob, who both want to push some commits to the same repo. If you’ve used Git, you may know that only the first person to push their commits will succeed; the other person will have to rebase on the first person’s commits and then try again. So if Alice and Bob both try to push their commits at the same time, GitHub has to decide who goes first (their commits will be accepted as-is) and who goes second (and must rebase). Alice and Bob might be connected to different servers, though. How are those two servers going to discover Alice and Bob’s conflict, resolve it amongst themselves, and present a single consistent timeline of events that is the same for Alice, Bob, and everyone else using the repo?

This situation sounds a little like the restaurant example from above. We have a group (of servers this time, rather than people) with a disagreement: one thinks Alice’s commits should be merged first, the other thinks Bob’s commits should be merged first. Nobody really cares which commits go in first; whoever goes second can easily fix the problem by rebasing and merging again. But it does matter that everyone see the same order of commits, so the servers do need to reach agreement before any commits can be merged. Which means they’re stuck on the decision. Sounds like just as much a consensus problem as people trying to pick a restaurant!

### Key-Value Store

A key-value store is a type of database, basically an implementation of the ‘map’ data structure stored on servers and made accessible over the network. For large enough datasets, one server might not be big enough to store the whole map, so it’s common to support splitting a key-value store across multiple servers.

Say Alice and Bob are again connected to different servers. They both change a system config setting, causing both servers to try to update the same key in the same key-value store at the same time. Now we again have a disagreement: the two servers want to update the same key to different values. Once again, it doesn’t really matter whether Alice’s config update is accepted or Bob’s: whichever update is rejected, that user can look at the new setting value and make a decision from there whether to leave it or change it. But it is important that all servers are using the same config! We can’t have some servers using Alice’s config and others using Bob’s at the same time, so we have to pick one key-value update to apply and make every server agrees which one was applied. Another consensus problem.

### Lock Service

A distributed lock service implements the networked version of the mutex locks you may have encountered in multithreaded code. A thread which obtains a distributed lock can be certain no other thread on any server in the network also holds the lock at the same time. Locks in a distributed lock service often come with a timeout, so that the lock can be released automatically if a server crashes before releasing its lock. A lock service like this can be useful for a variety of things; for example, the key-value store from the previous example might use this service to lock keys for update, or assign a group of keys to a single server so that server can manage all the keys using plain (non-distributed) locks. 

Once again, say Alice and Bob are connected to different servers. This time they're both trying to open the same doc on a writing app. The writing app internally holds a lock on a doc while it’s being edited, so Alice's and Bob's servers now both try to obtain the same lock at the same time. By definition of a lock, they can't both take the lock at the same time, so the service needs to guarantee one server gets the lock, one doesn't, and both agree who has the lock. Once again, however, it doesn’t really matter which user gets the lock; since again we find our servers needing to reach agreement so they can move forward. Consensus strikes again!

## Consensus Problems

We have seen three examples where servers can get into disagreement with one another. In general, this seems to happen when you have multiple servers, shared state, and two updates to that state happening on two different servers at the same time. If those two updates aren’t compatible with one another, the servers end up in conflict. When this happens, we need a way to resolve that conflict so we can move on and continue processing the user’s request; since we need agreement to move forward, we call these conflicts *consensus problems*. Algorithms we can use to resolve these conflicts are accordingly called *consensus algorithms*.

In a way, the kinds of conflicts we’re dealing with here are inherent to the kinds of systems we’re trying to build. A distributed service may be made up of a network of servers, but users don’t have to worry about the way the network is laid out or what each server is doing; they treat the entire network as a single cohesive service and interact with the service as if it were running on one computer. When the user expects there to be one state (the service’s state), but the state is managed by multiple servers, conflicting updates are basically inevitable. Resolving these conflicts requires consensus algorithms, which perhaps is why people consider consensus to be foundational algorithms for the field of distributed systems.

> Despite this, there is a way to build these services without using consensus algorithms: design the system so that parallel updates can never conflict! This is the idea behind a class of data structures called [conflict-free replicated data types](https://en.m.wikipedia.org/wiki/Conflict-free_replicated_data_type) or “CRDTs.”

Even though problems of consensus are pervasive in distributed systems, the average dev rarely needs to employ full-blown consensus algorithms like Raft and Paxos; few devs ever need to implement them. That’s because consensus can often be worked into higher-level primitives like databases and other cloud services. These things ultimately are built using a classic consensus algorithm like Raft or Paxos internally, but also themselves provide consensus, freeing you from having to deal with consensus yourself. For example, in our examples above, it would be possible for GitHub to provide consensus on branch state by storing it in the key-value store we discussed, and it is in turn possible for the key-value store to use the our distributed lock service to provide consensus by taking locks on keys. The lock service itself would still need to use a classic consensus algorithm like Raft or Paxos, but then the key-value store can lean on the lock service to provide key-value consensus, and GitHub could use key-value consensus to provide GitHub repo consensus. Sometimes it takes a little digging, but there’s a Raft or a Paxos or some other consensus algorithm to be found in pretty much every distributed service.

## Fault-Tolerant Consensus

So far, consensus algorithms don’t sound very hard to design. There are lots of simple heuristics for resolving conflicts in a consistent way: for example, all nodes could pick the update that has the earliest timestamp, or have some way to sort the nodes and pick the update that came from the node with the lowest ID. Sadly, the only reason consensus sounds simple so far is because we’re missing an entire dimension of the problem. Let's talk about reliability for a second.

Think about the device you're using to read this book; have you ever had weird little problems with it? Freezes, or crashing apps, weird glitches, system-wide slowdowns, overheating, unexplained network disconnects, blue screens, anything like that? I’m guessing these things have happened to you, even if they don't happen often enough to be a major disruption day to day.

But now imagine you were using not just one device, but a thousand of them. Or imagine you’re a cloud provider, running hundreds of thousands of these things across the world, connected by thousands of miles of network cables. How often do you think you'd be dealing with these kinds of little problems? Heck, you'd probably never be able to fully rid yourself of them, no matter how hard you tried! Rare problems, multiplied by thousands of machines, become common. If you serve a million requests every minute, you should expect every possible one-in-a-million problem to happen roughly once every minute (over 1,400 times a day). There’s no way to squash all of these problems, so code running on this environment &mdash; and, by extension, you &mdash; will have to live with them. These little problems are called **faults**, and software that works despite these faults is said to be **fault-tolerant**.

Consensus algorithms that can be deployed in practical, real-world settings need to be fault-tolerant. Designing fault-tolerant consensus algorithms turns out to be hellishly difficult. These algorithms need to provide perfect, exact guarantees, but run on an imperfect platform that routinely fails to provide its stated guarantees. How does one do that? It’s something we’ll have to tackle later on.

## Properties of a Fault-Tolerant Consensus Algorithm

To finish our discussion about the problem space, let’s nail down a set of properties any useful consensus algorithm should have. Unfortunately for us, there isn't a well-accepted set of consensus properties we can rattle off here: database people have ACID (*atomic, consistent, isolated, durable*), but there's no similarly catchy acronym for consensus algorithms. We'll have to wing it.

This is one of those rare cases where I think it’s a good idea to use big fancy words to describe our ideas. Each of these properties will mean something specific, and we’ll be referring back to these a lot as we try to design a working consensus algorithm, so it’d be useful to have some nice short 1-2 word names for our consensus properties, even if that means we end up having to use big words for little ideas.

### The Consensus Properties

Remember that our definition of consensus was “agreement that lets a group move forward.” Fundamentally, consensus algorithms take a set of conflicting options, decide on one, and guarantee that decision cannot be undone. This allows the decision to be treated as final, which in turn allows calling code using the consensus algorithm to move on and act on that decision. For example, deciding Alice’s commits will be merged first allows GitHub to tell Alice her commits were merged and Bob that he needs to rebase.

Lets unpack that. I see three separate properties that together should cover what we just said:

**Conflict resolution**: The algorithm gracefully handles conflicts when they arise; for example, it’s not an error for multiple servers to try to obtain the same distributed lock at the same time, even though the lock can’t be granted to all the servers simultaneously. The algorithm does something to resolve the conflict (in this example, choosing which server gets the lock).

**Coherence**: Every server can eventually find out what decision the algorithm made. In other words, all servers get the same return value when the local copy of the consensus algorithm finishes. Continuing with the lock service example, coherence would mean every server agrees which server got the lock; no server erroneously believes some other server got the lock.

**No-Decoherence**: Once consensus is reached, the decision is final. No matter what new information becomes available in the future, it won’t change a decision the consensus algorithm has already made. This is what makes it safe for calling code to act on the decision. If we didn’t have this, consensus would not be useful for our lock server, because the consensus algorithm could change its mind and decide a server no longer holds the lock *after* that server has already started running the lock-protected code!

To safely treat decisions as final, we need to be pretty strict about the no-decoherence property. A consensus algorithm must ensure no server *ever* sees the wrong result, even while the algorithm is still running and hasn’t completed yet. Equivalently: the instant *any* server can see a decision has been made, it must be impossible for the decision to change. The step that first makes a decision visible must also lock that decision in forever, as one atomic operation.

I’ll call the three properties together the **consensus properties**, since they fall directly out of the definition of a consensus algorithm. But there’s still one more property we need:

### Fault Tolerance

A consensus algorithm that can actually be used in a real production environment must tolerate real production faults, because it’s just not feasible to get the entire network running perfectly and keep it that way. A consensus algorithm should be resilient to things like:

* Hardware failures
* Server crashes
* Power loss
* Servers taken down for upgrade
* Network disconnects
* Slow networks
* Lost network messages

No matter how many of these things are going on, the algorithm must never violate any of the consensus properties (conflict resolution, coherence, no-decoherence) mentioned above; however, it’s acceptable for the algorithm to proceed slowly or even halt if the system is in bad shape and the fault rate is really high. At the limit, this is unavoidable anyway; if every server loses power, your code ain’t running, so good luck coding a workaround! To recap, it’s okay if the algorithm eventually stops working due to faults in the underlying system, as long as it never starts doing the wrong thing.

Throughout this book, when evaluating an algorithm’s fault tolerance capabilities, we always consider the absolute worst possible case. For example, if we want to claim an algorithm can tolerate one server crashing, then it must be true that *any* node can crash without halting the algorithm. If, for example, most servers were allowed to crash but there was one ‘special’ server that the algorithm cannot allow to crash, then we’d technically say the algorithm does not tolerate a single crash, because in the worst case, that special server could be the one crashed. This might seem harsh, but keep in mind that even rare situations &mdash; like that one special server crashing &mdash; become common at high scale.

### Recap

As we move into the design phase of this book, keep these properties handy; we’ll be referring back to them a lot:

> **Conflict Resolution**: When conflicting updates are proposed, the algorithm picks one and rejects the others
>
> **Coherence**: At the end of the algorithm, every server agrees which update was picked
>
> **No-Decoherence**: The instant a decision is made, it is final. New information cannot change committed decisions.
> 
> The three properties above are referred together as the **consensus properties**
> 
> **Fault Tolerance**: The system degrades gracefully when there are faults in the underlying system. Faults do not ever cause the algorithm to violate a consensus property.

<center>
  <a name="part2"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    2: Designing a Consensus Algorithm 
  </h1>
</center>

TODO you don’t understand a piece of code until you try to change it, and you don’t understand an algorithm until you try to design it. Best way to start is just start trying stuff, it probably won’t work but you’ll figure your way around the problem space as you go

At this point our knowledge of consensus algorithms is rather textbook; let’s get more practical by inventing a working algorithm ourselves. In this section we’ll try out a few ideas, just to play around and get a feel for the problem space. The ideas we come up with in this section will be good, but incomplete. We’ll find ourselves repeatedly hitting a dead end. We won’t be able to get our designs in this section fully working until part 3, where we figure out what the dead end is and how to get around it.

## A Few Restrictions

TODO justify

TODO binary decision, red vs blue

TODO one-shot decision

TODO no-preference / arbitrary conflict resolution


---
TODO starting from here we have old content from before the intro and properties section rework. It should be mostly fine but there are probably small details that need updating
---

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

---
TODO a takeaway we should add here: a lot of things that look like solutions to the consensus problem are themselves things that rely on consensus ... e.g. consensus is easy if you have reliable leader election, but reliable leader election relies on consensus so we can’t use it. So any time you say “does this solve the consensus problem,” first ask, “does this depend on a solution to the consensus problem?” Can you find a conflict that must be resolved globally for the algorithm to function correctly? For leader election that conflict is “who is the leader?” On second thought, this might just be an entire new reflow of this heading ... 
---

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
  <a name="part3"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    3: FLP
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
  <a name="part4"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    4: Paxos
  </h1>
</center>




