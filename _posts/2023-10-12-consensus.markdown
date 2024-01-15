---
layout: post
title: Consensus Algorithms
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

Just as the best way to understand a piece of code is to change it, the best way to understand an algorithm is to try designing it yourself. In this chapter, we’ll try to design a fault-tolerant consensus algorithm. We won’t be able to get all the way there in one chapter, but we’ll learn a few important things along the way.

## A Couple Restrictions

To keep things from getting too complicated too fast, let’s start out by placing a couple of limitations on our design. The goal is to make it easier to get to a working consensus algorithm to begin with, while ensuring there’s still a path from the more basic thing we start with to the more advanced thing we want to end up with.

First, we’ll start by making a **binary decision**: our algorithm will be designed to handle exactly two conflicting updates, and the algorithm needs to get all nodes to agree on one of the two. In real life, we’d probably want an algorithm to handle arbitrary many conflicts, but if we start by handling two, it’s very likely we’ll find a way to extend the algorithm from two conflicts to N conflicts. 

To make it easier to talk about the two conflicting options, we’ll also give them names: the <span style=“color:red”>red</span> option and the <span style=“color:blue”>blue</span> blue option:

[diagram introducing the two colored circles we’ll use throughout the book]

Remember, these options can stand in for anything else: 0 and 1, yes and no, apples and oranges, etc. 

Furthermore, we’re going to restrict ourselves to building **one-shot** consensus. The algorithm will be able to resolve conflicts one time, and never change it. This is too simplistic for real-world use; for example, it would mean the lock server would be able to grant a lock to one thread one time, and never release it. But if we can build a one-shot primitive, we can find ways to implement a sequence of updates by using a stream of one-shot decisions.

So, in conclusion: we’ll design a consensus algorithm that picks one of two options, one time, and sticks with it forever. And we think we’ll be able to extend that into an algorithm that works with any number of conflicts, any number of times.

## A Programming Interface

To nail down the design further, let's think about what kind of API we want give to clients using our consensus algorithm.

If we want the conflict-resolution property to really make sense, we need to collect proposals separately from deciding on which proposal to accept. So we probably want to split our interface into two methods:

* **propose()**: offers the consensus system a value you'd like to change the state to, which can either be “red” or “blue”
* **query()**: ask the consensus system what value the state actually was changed to

Code that wants to update the shared state first calls **propose** with the value it prefers, followed by **query** to see what value was chosen. Code that just wants to read the shared state can skip **propose** and just call **query**. The consensus system guarantees every **query** call returns the same value, as required by the coherence and no-decoherence rules.

For example, we could make the lock server example work by defining a shared consensus variable called `owner`, which is defined as:

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

Make sense? Then let’s take our first stab at a design:

## A Leader-Based Algorithm

Let me propose a basic design:

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
* **fault-tolerance**: hmm ... we have a problem here.

At least the way we've designed the algorithm so far, it would seem the leader is a **single point of failure** &mdash; if it crashes, or loses power, or gets disconnected from the network, or any number of other bad things happen to the leader, nobody else is going to be able propose or query the consensus variable. Even one fault halts the entire algorithm; so the algorithm is not fault-tolerant.

But maybe we can rescue this design? What if we maintain backups of the leader’s state, and promote one of the backup nodes to leader when the leader fails? It’s a good idea, save for one major problem:

## Appointing Leaders is a Consensus Problem

You see, promoting a backup node to leader only works if all nodes in the network agree that backup node is now the leader. If some nodes think one node is the leader and other nodes think another node is the leader, there are two leaders in the system. If the two leaders get out of sync, then the two sets of nodes listening to the two leaders start returning different values from their **query()** method, and the coherence property is violated. 

Consider the following network. Node 1 is currently the leader; nodes 2-5 are following. Nobody has made a proposal yet, so the current consensus `value` on the leader is `null`:

[diagram]

Now let's say the network faults, splitting the network: now nodes 1-2 can talk to each other, but not to nodes 3-5; likewise, nodes 3-5 can talk to each other, but not nodes 1-2. Our system has now split into two sub-networks:

[diagram]

This situation, where groups of nodes can talk to other nodes within a group but not the rest of the network, is called a **network partition**. It can happen in practice, for example, if a network box which connects the two groups of nodes gets unplugged or something.

Well, here's the problem: node 1 was the leader, and nodes 3-5 cannot talk to it; for all they know node 1 might have crashed. So they fail over to a new leader. Let's say they pick node 3:

[daigram]

Uh oh, now there are two leaders! And now that there are two leaders, they can decide on different things. Say node 2 proposes <span style="color:blue">blue</span> and node 4 proposes <span style="color:red">red</span>. Then the different sub-networks end up with different accepted proposals:

[diagram]

Now a client that queries the system can either receive <span style="color:blue">blue</span> or <span style="color:red">red</span>; clearly the coherence property is not upheld.

This situation, where a system expects to have one leader but actually ends up with two or more, is called **split-brain**. To prevent split-brain, we need a way to get all nodes to agree which node is currently the leader node.

Alas, getting all nodes to agree which node is currently the leader is a consensus problem. If you’re trying to solve the consensus problem in the first place, you can’t rely on having the problem already solved! So if we don’t know how to make a consensus algorithm, we don’t know how to safely promote a backup node to leader either. We can’t use this approach to design our algorithm.

But we did learn an important lesson: a lot of ideas which look like potential solutions to the consensus problem, themselves rely on a solution to the consensus problem. So if we ever find an easy answer, we should first check if the solution is easy only because it assumes an existing solution of the very consensus problem we’re trying to solve.

So where does that leave us? The starting example I chose didn't work out, but we did learn something important in the process: we need to design a **peer-to-peer** algorithm. Instead of putting one node in charge, we need to set things up so nodes cooperate as equals, haggling out which proposal should be accepted.

Do you know of any way to do that?

## Second Stab at a Design

I'll bet you’ve used peer-to-peer consensus algorithms in real life:

Imagine the restaurant example once again. What exactly happens once you realize you need to decide on a place to go? Maybe someone throws out an idea, someone else throws out a different one, someone agrees, someone disagrees. Before you know it, a group opinion has formed. Book, consensus reached!

Kinda sounds like voting to me ... do you think we could make an algorithm out of that?

## The Majority-Rules Voting Algorithm

We can totally code up something that throws out proposals and votes on them, just like in the restaurant example above. But unlike real life, where people have preferences among options, we’ll code an algorithm where each node votes for whichever option it heard about first.

Here's a sketch:

Every node in the network will have its own local accepted proposal variable, which will keep calling `value`. Initially, `value` is null. When client code calls our **propose()** method, proposing red or blue, we send that proposal to all nodes, including ourselves. When a node receives a proposal, it checks its local `value` variable; if it’s still null, then this is the first proposal this node has received, so it sets its local `value` variable to that proposal, “voting” for it.

A node can only vote for one value, and can never change its vote once set &mdash; so if a node votes for someone else's value, and gets a local **propose()** method call, nothing should happen, because this node already voted for something else and cannot change its vote.

Finally, we implement **query()** by tallying votes: ask every node what it voted for (what its current `value` variable holds), and see if any proposal has been voted in by a majority, more than half of the nodes. If so, that is the consensus decision; otherwise, consensus is considered to be not yet reached.

One problem we’ll need to deal with is split votes: what if every node votes, but no proposal reaches a majority? For now we’re only letting clients propose red or blue, so there are only two possible proposals; if we have an even number of nodes, it could be that each proposal ends up with exactly half the votes. At that point, our consensus algorithm is stuck: there is no majority proposal, and there are no more votes, so there’s no way to make further progress. But we can fix this by requiring the algorithm to run on an odd number of nodes; that way when all the votes are in, no matter how the votes work out, either red or blue must have gotten more than half the votes.

Here's the algorithm again, as pseudocode:

```
consensus {
  value: Proposal; // the accepted proposal
  peers: Node[]; // all nodes in the network
  
  init {
    value := null // no accepted proposal yet
  }
  
  // this is the propose() API
  propose(proposal) {
    // peers.all includes this node as well
    peers.all.send(proposal)
  }
  
  // received a proposal from a peer
  on peer proposal(proposal) {
    // accept the first proposal, ignore others
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

So ... does *this* design work?

* **coherence**: ✅ &mdash; only one proposal can reach a majority, and that's the proposal **query()** always returns
* **conflict resolution**: ✅ &mdash; the voting system allows for multiple proposals and decides which will be accepted
* **no-decoherence**: ✅ &mdash; nodes cannot change their votes, so once a proposal reaches majority, it cannot lose its majority. Vote counts are like stonks, they always go up
* **fault-tolerance**: . . . 😀 I told you this would be tricky, didn't I? 

No, this algorithm isn't fault-tolerant either! But we're getting better at this &mdash; this time, the problem is much more subtle.

This design is pretty resilient, but depending on how things play out, it sometimes has a **window of vulnerability** where one extremely poorly timed crash could bring the entire system to a halt, preventing a majority from being reached until that computer is brought back online. This case might be rare, but it still happens as a result of just one crash, so by worst-case analysis we technically have to admit this algorithm cannot withstand just one crash &mdash; hence, it's not fault-tolerant.

More specifically, the problem is that one crash leaves us once again with an even number of nodes, so split votes become a problem again. Let's watch this in action:

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

Or maybe it actually hadn't made a decision yet, and it's safe to tiebreak some other way?

[diagram with 3 red, 3 blue, one blank in a thought bubble]

If we want to move on without the node that crashed and still uphold the no-decoherence guarantee, we really need to know whether the crashed node voted, and if so what it voted for. But the node is gone; it can't tell us what it already did. In fact, it might even be gone for good; what if it went offline because it caught on fire. Things happen in data centers, you know?

We're good and stuck now, up until we manage to get the node back up and running from where it left off. Which may not be possible. Because of fires.

To conclude, the loss of a single node with this algorithm is usually just fine, but in cases like the above it can lead to a split vote, and then we’re good and stuck. So sadly, this algorithm also is not fault tolerant.

---

## Intermission

If you have the time, this would be a good point to step away from your computer and think over the above. What went wrong with our two approaches above? Can you think of a way to fix either one? Or a different strategy entirely?

This is the point in the thought process where the field as a whole got stuck, for years. People kepts trying to find ways around this problem, or different solutions that don't have this problem, and failed to find a way through, repeatedly. Based on that, I'm guessing you won't copme up with a working solution in an afternoon walk through the woods. But think through a variety of approaches and seeing them not play out might help build your understanding of the problem space, and what we'd have to figure out how to do if we wanted to make a fault-tolerant consensus algorithm.

If you take me up on this exercise, two things to keep in mind:

1. If you think you have the solution, remember to do an absolute-worst case analysis: like with the voting example above, if there's one specific execution where the loss of one specific node at the exact wrong time blocks the algorithm from making progress, it's not fault tolerant. Sometimes, finding that one situation is tricky!
2. Be wary of solutions which appear to work, but actually rely on some other form of consensus. For example, in our single-leader algorithm, we found that getting all nodes to agree on a leader is itself a consensus problem, and thus we couldn't rely on safe leader election to design a consensus algorithm in the first place.

Of course, for the curious and the impatient, you can also just read on. With that, it's adieu for now, perhaps!

---



<center>
  <a name="part3"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    3: FLP
  </h1>
</center>

---

When I started writing this I was misremembering how FLP works. But I think most of what’s already here is salvageable. 

Basic flow:

* Intro idea of setting up the system so that, once a specific message has been processed, the system will have decided
* Maybe at this point preview that the node which receives that message is a SPOF. 
* Show the FLP lemma 3 mechanics on a simple 3 node majority voting setup. Show how “vote red” and “vote blue” sent to the undecided node always result in a system which has decided. Proceed to show the algorithm deadlocks if that node crashes, and any attempt to recover from this state leads to split brain in the case where the undecided node is alive, but slow
* Revisit the same mechanics abstractly for any algorithm. A good way to explain the ordering problem and fault detector is to imagine the case where the node fails and sigma must now recover, then imagine an alternate universe where everything happened exactly the same except p didn’t crash, it’s just running slow. (Alternate universe is the key here). There’s no way to tell the difference but sigma must do something different in this case (not decide, yield the decision to p). 
* Walk through the FLP main proof, which takes an algorithm that never forces a decision and finds a way keep to delivering the oldest message to each node round robin, forever, without ever making a decision
* Maybe mention fault detection oracles. 
* Conclusions: 
* 1. If we ever can say our algorithm terminates, we should be suspicious
* 2. If it appears the system has always decided after processing a message sent before the system has decided, we had better look for deadlocks or split brains
* 3. As they mention in the paper’s conclusion, probability of termination might suffice
* This all loosely suggests  best effort retry loop. Wonder how one does this for majority voting 

Maybe we don’t need to mention decision points at all. The only crux of the proof is “forcing a decision” which is a bit different than the direction I was originally going.

On a related note, the properties we described before no longer lead into the discussion I want to have next, so we don’t need them per se. And also, if we do want to use properties, the standard ones appear to be agreement, validity and termination (with the care to somehow specify no-decoherence implicitly, eg by a write-once output register)

Maybe we could restructure the previous chapters not to have a set of properties (or have the more basic A/V/T set from literature) and then in this chapter introduce one last example that shows why the sigma can’t recover in the faulted case and also avoid split-brain in the no-faults case. The last example is “majority voting with tiebreak” and you show how it works with exactly 3 nodes in the network.

Also jump off with a discuss about what “a message sent before the system decides, system decided once processed” really means in an intuitive sense, and how it’s related to termination. 

---

Let’s try a flow like this:

1. Current first section as is, introducing the idea of proving something doesn’t work because we keep running into the same old dead end
2. What doesn’t work is (message from lemma 3, stated in the paper’s precise yet somewhat obtuse manner)
3. What the heck does that even mean? Do a majority voting 3-node example and point out the votes that do that
4. Why does this case show up? Preview the idea of an algorithm that terminates has a finite number of messages. If you’re going to stop sending messages, one of those messages had better be guaranteed to make the decision, or you exit with no decision 
4. What’s wrong with having such a message? The system cannot tolerate even one crash fault: sending the message is a point of no return, it is now the recipient’s job to decide based on its local message delivery order. But if the recipient chooses that moment to crash, the decision will never happen
5. Back into the example, show the algorithm deadlocks if the undecided node crashes and the algorithm does nothing else
6. But maybe we can do something else; propose a tiebreak where red always wins (chosen fairly by asking my toddler what his favorite color is)
7. Problem: the alternate universe where everything happens the same way and we get into the exact same state, except the undecided node didn’t crash, it was just being slow. Now we have two decisions, split brain
8. Okay, so if the undecided node crashes we get into a deadlock, and any attempt to make a decision after the crash leads to split brain. Neither is acceptable, but we’ve exhausted every option here. It seems we’re at a total dead end
9. This is compelling: this algorithm has the message FLP predicts is problematic, and indeed we hit a dead end. FLP shows this dead end is total, as long as we have that weird message this exact problem always occurs, using the exact same line of reasoning 
10. Retrace the lemma 3 proof, this time using closer to the FLP notation, but constantly referring back to the example to keep it concrete
11. Retrace the main FLP proof using something closer to their notation as well
12. Conclude you can either design an algorithm which has the weirdly defined message and guarantee termination (but not work in spite of one fault) or you avoid that message and thus work in spite of one fault (but never terminate)
13. Segue out by saying, it might seem we just proved it impossible to make a practical consensus algorithm, but actually there’s a little wiggle room left. In fact, it’s even mentioned in the FLP paper at the end. Pull the quote. Restate as non-guaranteed termination is very different from guaranteed non-termination!
14. Wax poetic about randomization and symmetry breaking. Sure, the bad thing could always keep happening, but what are the odds it keeps happening so many times?
15. State the strategy is to avoid  the message, thereby be able to withstand at least one fault, and deal with the non-guaranteed termination by making the probability of non-terminating fall rapidly with each iteration of the algorithm 

---

## When the Going Gets Tough, the Tough Prove the Going's Pretty Darn Tough ... and Give Up

Before a working consensus algorithm was discovered, people chewed through this problem just as you might have during the intermission above. And they kept running into the same dead end, over and over. They could make an algorithm that provided all the consensus properties, and even still make it *usually* fault tolerant, but there'd always be that one case, one little window of vulnerability where one node crashing brings the entire algorithm to a standstill.

Some advice: when you're trying to solve a problem, and no matter what you do, you keep running into the same dead end, the next thing you should try is to prove impossibility: show that dead end will always come up no matter what you do, due to something about the problem space. That proves the problem is actually not solvable, which will save you the time of trying to solve the unsolvable!

Well, that's exactly what three researchers did in the mid-1980s. In their paper *Impossibility of Distributed Consensus with One Faulty Process*, Fischer, Lynch and Paterson (the "FLP" in "FLP result") explained exactly why nobody could come up with an always-fault-tolerant consensus algorithm. Their paper uses the language of formal mathematics to explain their ideas in the form of a proof, but tied up in all the state machines and network models and whatnot is a simple and insightful idea. In this chapter, we'll see what they saw.

## The Goal

A note for people who don't eat math for breakfast: we're about to build is an *impossibility proof*; we want to make the argument that there is no such thing as a fault-tolerant consensus algorithm.

To do that, we're going to talk about all possible consensus algorithms at the same time, using abstraction. Just as you can take objects like "Car" and "Boat" and "Airplane" and unite them under some abstract "Vehicle" interface so we can make code that works for all possible vehicles, so too are we now going to abstract away the details of actual consensus algorithms so we can make a statement about all consensus algorithms abstractly. And the statement we're going to try to make is, all consensus algorithms have a way of getting stuck: there's at least one execution where a single node crashing is enough to bring the algorithm to a standstill and prevent the system from ever reaching consensus.

Working this way makes it easy to get lost in abstraction, so along the way if you find yourself lost, try to map what we're talking about abstractly to a real consensus algorithm you already know, such as the leader-based replication and majority voting algorithms we talked about in chapter 2. With that, let's get cracking!

## Decisions, Decisions

To understand the FLP result, first we need to adopt Fischer, Lynch and Paterson's unique way of looking at all possible consensus algorithms abstractly. Take another look at what we've been calling the consensus properties:

> **Conflict Resolution**: When conflicting updates are proposed, the algorithm picks one and rejects the others
>
> **Coherence**: At the end of the algorithm, every server agrees which update was picked
>
> **No-Decoherence**: The instant a decision is made, it is final. New information cannot change committed decisions.

Together, these tell a kind of story about how consensus algorithms work. Conflict resolution says, at the start of the algorithm, there are two options (we've been calling them "red" and "blue"), and either could be the one the algorithm ends up choosing. Coherence says, at the end, there is only one chosen option: red or blue. And no-decoherence says the decision is made in some way that also instantaneously locks it in, ensuring it can never be undone.

If you think about it, this means the story has a climax: there must be a single step of the algorithm that makes a decision once and for all. Before that step runs, the system could still choose either red or blue; after that step, all fates are sealed, and either red or blue has been chosen. We'll call this step the **decision point**.

Since every step of a distributed algorithm runs on a single node, and a decision point is just a special name for a step of the distributed algorithm, we know the decision point is a single instruction &mdash; a single line of code, basically &mdash; running on a single node. In the leader-based replication algorithm, that line is easy to find: it's the point when the leader receives the first proposal and assigns it to its local `value` variable, here:

```
  on client proposal {
    // accept the first proposal, ignore others
    if (value == null) {
      value := proposal   <-- decision point
    }
  }
```

For majority voting, it's a little trickier to find. The decision point for majority voting is when one proposal receives a majority of the nodes' votes. That line of code was here:

```
  // received a proposal from a peer
  on peer proposal(proposal) {
    // accept the first proposal, ignore others
    if (value == null) {
      value := proposal  <-- vote registered
    }
  }
```

Except that line of code is only *sometimes* the decision point!

* Sometimes a node votes, but even after doing so, no proposal has a majority yet. So the vote is insufficient for making a decision
* Other times a node votes after a proposal already has a majority. So the vote is redundant
* Only one vote causes a proposal to cross the critical boundary sub-majority to majority. This vote locks in the majority, preventing other proposals from ever reaching a majority. So that vote is the algorithm's decision point.

So maybe we should amend our definitions so that registering a vote counts as a decision point too. Let's say this:

> A **decision point** *potentially* makes a system wide-decision. However, it may be **insufficient** (executing it does not cause the system to reach a decision), or **redundant** (the system had already made a decision before it executed).

That's compatible with the way the FLP paper analyzes consensus algorithms: as a set of potential decision points glued together by code that transmits messages on the network, disseminates results, and so on. Every consensus algorithm has at least one decision point, and the decision points are the interesting parts of the algorithm.

At first glance, it might seem like one could make a consensus algorithm fault tolerant by setting up many decision points running on many nodes; that way, if a node crashes without executing its own decision point, other decision points executing on healthy nodes still have a chance to make the final decision. But we tried this in the majority voting example and it didn't work: there was still a way losing just one node could leave us with a split vote, unable to make progress. Through the FLP looking glass, we'd say the loss of just one potential decision point 

But is that a problem with majority voting, or more generally a problem with having a limited number of decision points? According to the FLP result, it's the latter: any algorithm that has a finite number of decision points can terminate without making a decision if just one node crashes. Let's see how . . .

## The FLP Result

FLP gives us a procedure we can use on any consensus algorithm to get it stuck, unable to make a decision, as long as it has a fixed number of decision points. Here's all you need to do:





---

TODO: wait ... not only did we fail to justify the sequentializing aspect before, but also throwing away the superfluous ones is also nonsense if not justified! People might add redundant decision points specifically to handle faults, so it doesn't make sense to just ignore them on the basis of them not needing to do anything, right?

Maybe it's time to reread the paper.

---





First, analyze the algorithm and find its decision points. If present, throw away any 'superfluous' ones that are always redundant &mdash; ones that are guaranteed to execute after the 



 list its decision points. If there are any 'superfluous' ones that are always redundant (they never decide anything), remove them. We should be left with a subset of the algorithm's decision points that can actually 











---

First, take the algorithm's set of potential decision points. If there are any superfluous ones that cannot decide anything because they are always redundant, remove them. We are left with the set of potential decision points that can actually make decisions.

Next, find an execution of the algorithm which uses every potential decision point, not making a decision until the very last one. In other words, find a way the algorithm can play out where no potential decision point is redundant: all but the last one run without making a final decision, and then the last one makes the decision. We know such an execution of the algorithm exists, because none of the potential decision points is superfluous.

For this execution consider the case where every potential decision point except the last one executes. By construction, we know the algorithm has not decided yet. Now assume the node which runs the final potential decision point crashes. The system has not decided, and now there are no potential decision points remaining. The algorithm now has no choice but to terminate without deciding. Since the algorithm failed, but there was only one fault (one node crash), we conclude the algorithm was not fault-tolerant.

---













---

TODO reformulate the below, not as "adding a termination guarantee messes everything up," but rather as "any consensus algorithm with a finite number of decision points can fail this way"

Also, I reformulated above so we can say "decision point" instead of potential decision point, because that's a mouthful and because I don't want to break out the TLAs.

---













We'll show that any consensus algorithm that provides *Conflict-Resolution*, *Coherence*, *No-Decoherence* and *Termination* is not *Fault-Tolerant*: it can fail by exiting without making a decision in the event just one node crashes. We'll show this happens by virtue of the properties themselves, not any detail of how the algorithm implements the properties, thus showing *any* algorithm which correctly implements termination-guaranteed consensus cannot be fault tolerant:

First, as we discussed earlier, we know from the algorithm's *Conflict-Resolution*, *Coherence* and *No-Decoherence* properties that it must have a "decision point," which is a step running on a single node that decides once and for all what value the system will decide on.

If the algorithm has just one decision point, we know it is not *Fault-Tolerant* because the node that runs the decision point can crash. However, it's possible the algorithm provides many *potential* decision points so that, if one potential decision point never executes due to a crash, other ones running on healthy nodes can potentially compenstate for the failure.

However, due to the *Termination* guarantee, we know there is a finite supply of potential decision points. Knowing this, we can use the following procedure to find at least one way the algorithm can exit without deciding, while only being asked to tolerate a single node crash:

First, take the algorithm's set of potential decision points. If there are any superfluous ones that cannot decide anything because they are always redundant, remove them. We are left with the set of potential decision points that can actually make decisions.

Next, find an execution of the algorithm which uses every potential decision point, not making a decision until the very last one. In other words, find a way the algorithm can play out where no potential decision point is redundant: all but the last one run without making a final decision, and then the last one makes the decision. We know such an execution of the algorithm exists, because none of the potential decision points is superfluous.

---

TODO: why is the last sentence of the above paragraph true?

Like you can do silly degenerate things like say if nodes 1 and 3 vote red, that counts as two extra votes for red. 

---

For this execution consider the case where every potential decision point except the last one executes. By construction, we know the algorithm has not decided yet. Now assume the node which runs the final potential decision point crashes. The system has not decided, and now there are no potential decision points remaining. The algorithm now has no choice but to terminate without deciding. Since the algorithm failed, but there was only one fault (one node crash), we conclude the algorithm was not fault-tolerant.

None of this was specific to any one algorithm; the procedure above works for any algorithm that provides *Conflict-Resolution*, *Coherence*, *No-Decoherence* and *Termination* properties, just by virtue of providing them. That means, *any* algorithm that provides these properties is not *Fault-Tolerant*. &#8718;

That's the complete FLP result; but if you're like me, thinking abstract about all possible consensus algorithms simultaneously hurts your brain; it certainly hurts mine. So let's take a page out of [Eugenia Cheng's book](https://www.hachettebookgroup.com/titles/eugenia-cheng/beyond-infinity/9780465094820/?lens=basic-books), and recap the proof as a strategy game. Allow me to present . . .

## The Crashing Game

Here are the rules:

* I will present you with a consensus algorithm. I won't tell you what it is in advance, but I'll claim it provides 
* Your goal is to crash the algorithm, so that it gets stuck and never makes a decision
* You are not allowed to change the rules of my algorithm or edit any of its steps
* You do have complete control over all the indeterminate stuff, like the order network messages are delivered to nodes or the relative order of concurrent instructions running on different nodes in parallel
* You get to crash one node exactly one time. That node, once crashed, stays offline forever

You will always win this game if you play the FLP strategy. Here's all you need to do:

TODO finish this

TODO new heading or two to apply the FLP proof to each of leader-based replication and majority voting. We could make these examples of the crashing games as sub-heading: I present the algorithm, and then say your FLP strategy is to do this: yadda yadda

## Doing the Impossible

TODO segue out. Start by being a little distressed like, oh no, this is a book about fault-tolerant consensus algorithms and we just proved fault-tolerant consensus algorithms don't exist? Well, don't worry. Actually, we proved "fault-tolerant consensus algorithms that terminate" don't exist. We didn't say anything about consensus algorithms that don't terminate. I mean, it's far-fetched, but maybe we can make a consensus algorithm that doesn't terminate? Then hard cut to the "4: Paxos" heading below.

<center>
  <a name="part4"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    4: Paxos
  </h1>
</center>

---

Possible good hookup with FLP chapter, needs research since I don’t member the details. But I had the impression Lamport was trying to extend the impossibility result to encompass the partially synchronous case too. We can hook up by saying, basically, that he was trying to show the ability to detect timeouts doesn’t make the sigma steps actually work in both the faulted and no-fault scenarios. 

Intuition: you would need a way for red and blue to not only decide on a color, but also invalidate the third node entirely; that way, if it decides after the tiebreak is done its decision is already superseded. But that leaves a potential window between node 3 deciding and its result being invalidated by the other two nodes, potentially allowing the system to change a set decision, which is illegal.

But in trying to prove this always happens we instead would find an interesting idea: what if we had an algorithm that could atomically (a) prevent a previous vote from making progress and (b) determine what the previous result was. Well, you can’t actually 100% determine the result, but you can eliminate all but one possibility...

This is all completely in bounds with FLP, there is no message the system is guaranteed to have decided after processing, and there is no termination guarantee. But with a little bit of randomization, the real world odds of non-termination drop so rapidly it doesn’t really matter.

---



