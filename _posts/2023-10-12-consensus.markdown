---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

<div style="padding-left: 1em; border-left: .3em solid #eee;" markdown="1">

*"How can you make a reliable computer service? It may be difficult if you can’t trust anything and the entire concept of happiness is a lie designed by unseen overlords of endless [deceptive power](https://scholar.harvard.edu/files/mickens/files/thesaddestmoment.pdf)."*

</div>

How can you make a "distributed" variable? By that I mean: say I have a bunch of computers connected together by a network. Call each of these computers a "node." How can you make a variable that any of those computers can get or set? How do you share a single variable across a group of nodes?

DIAGRAM: nodes a network, thought bubble question mark in the middle for a variable

Well, we all know how to make regular, single-node variables. Let's pick one of the nodes and declare a regular old variable on that node. Call that node the "leader."  The leader can get or set the variable normally, since for the leader, it's just a regular old local variable. Let's also run an RPC server on the leader, and provide RPCs to get or set the variable. That way, other nodes can read and write the variable by sending get and set RPCs to the leader, respectively.

DIAGRAM: one node is highlighted as the leader; all others and sending get/set RPCs.

In the words, the leader allocates a variable and runs an RPC server like this:

```
leader {
  variable := null;
  
  get {
    return variable;
  }
  
  set(value) {
    variable := value;
  }
  
  on client get(request) {
    request.respond(variable);
  }
  
  on client set(request) {
    variable := request.value;
  }
}
```

All other nodes use a client that sends RPCs to the server, like this:

```
client {
  get {
    return leader.send_rpc(get);
  }
  
  set(value) {
    leader.send_rpc(set, value);
  }
}
```

So, did we do it? Do we have distributed variables now?

## Fault Tolerance

Think for a second about the device you used to open and read this webpage. Have you ever had weird little problems with it?  Freezes, or crashing apps, system-wide slowdowns, overheating, unexplained network disconnects, blue screens, anything like that? Hopefully these things don't pop up enough to be a signfiicant daily disruption, but even so I'm guessing you've run into these from time to time. How often would you say these kinds of things have happened to you? Would you ballpark the number in hours, days, weeks?

Now imagine instead of having to keep just your one device up and running, you had to manage a network of hundreds of them; or maybe you're a cloud provider managing tens of thousands of them. How often do you think at least one of your computers will have one of these problems? Heck, do you think you'll ever get to a point where *all* of them are working at the same time?

Welcome to the world of distributed systems! At scale, you'll find yourself dealing with rare problems all the time &mdash; even if a problem rarely affects any one computer, if you have a lot of computers, the odds any one computer is having that problem right now are pretty good. And since you're going to be deluged with these little glitches, freezes, disconnects, and so on, you have no hope of fixing them all and making them all stay fixed; and if you can't fix them, then the code running in that environment will have to live with them.

This makes distributed systems a kind of funny environment to work in. It's certainly unforgiving: the platform provides few guarantees &mdash; fewer than you'd expect &mdash; and even the guarantees you get on paper don't always hold up in practice. It’s a world where anything that can go wrong will go wrong, is going wrong, and has been going wrong for weeks unnoticed. Working on distributed systems is like playing a perverse game of Simon Says, where you make assumptions you think are already very conservative, only to find out that &ndash; Simon didn’t say! &ndash; that assumption can break too. As much as you'd like say the platform isn't supposed to do something, the fact always remains that it does, and the fact that it does is now your problem. In the end, it's always your job to make the system work, even though the platform you're running on frequently doesn't.

The kinds of hardware failures, software glitches and network problems we've been talking about are collectively known as **faults**. Software that continues to work in spite of faults in the underlying platform is said to be **fault tolerant**. Since there is no feasible way to eradicate faults in a large enough network, your choices as a distributed systems engineer are to make your code fault tolerant, or suffer frequent downtime and outages. That's no decision at all! Everything we code in a distributed setting must be fault tolerant; it's non-negotiable.

In that light, let's double-check our algorithm for making a distributed variable:

DIAGRAM: another copy of the RPC diagram, last one in the previous section

What happens if the leader node goes offline? Maybe it had an operating system crash, or someone tripped over its power cable, or [Ted the Poorly Paid Datacenter Operator](https://scholar.harvard.edu/files/mickens/files/thesaddestmoment.pdf) pulled the wrong network cable while trying to fix a different problem. What happens to the rest of the system if RPCs aren't reaching the leader node?

DIAGRAM: same diagram, but with the leader Xed out

Looks like we have a cascading failure on our hands. The leader failed, which is a problem, in and of itself, but on top of that, none of the *working* nodes can gets or set the variable either, because they rely on RPCs to the leader and the leader isn't responding right now. So whatever fault brought down the leader, the system is not tolerating. We need fault tolerance.

What should we try next? Oftentimes the solution to problems of fault tolerance and resiliency is **redundancy**: if the leader becomes unreachable, let's just fail over to a new leader.

## Leader Replication with Failover

Okay, so putting a variable on the network using RPCs isn't fault tolerant; we want fault tolerance, and our idea to add fault tolerance to our existing design (and thereby 'rescue' it) is to fail over to a new leader.

Of course, when a failover occurs, we don't want to lose whatever was in the variable before. But we also don't know if or when a failover will be needed. That seems to mean we need to eagerly store backups of the leader's variable on other nodes, so that we're ready to fail over to another node at any time.

So let's do that. To start, let's put a copy of our distributed variable on every node:

DIAGRAM

We'll still send get and set RPCs to the leader as before . . .

DIAGRAM

. . . Now, however, any time the leader sets the variable, it also sends an updated copy to all followers, telling each of the followers to update its backup copy of the variable:

DIAGRAM

TODO introduce the terms replica and replication

Now we have a tentative scheme for replication. Let's come up with a scheme for failing over.

(If you've worked on a distributed system before, you might have alarm bells going off in your head right now! Failovers are fraught with peril. A fully completed failover is a great thing, but a failover in progress is a rather sticky situation.)

To start, let's assign every node a numerical ID. It doesn't really matter how node IDs are assigned to nodes, as long as every node has a unique ID; say for example we require the operator to assign IDs to nodes via a config file or something:

DIAGRAM

Before our algorithm supported failover, the answer to the question "which node is the leader?" was a compile-time constant. It could have been hardcoded, or provided via a config file. Now that we support failover, "which node is the leader?" is answered by a runtime variable. We need a scheme for nodes to figure out who the leader is, at runtime. Also, whatever scheme we choose cannot itself rely on a leader &mdash; after all, the leader might have crashed, or gotten unplugged, or gotten disconnected from the network because Ted was in a rush to go home and watch his [extensive home collection of "Thundercats" cartoons](https://scholar.harvard.edu/files/mickens/files/thesaddestmoment.pdf).

Let's try to make a scheme where each node independently figures out who the leader is, using only information it already has. To begin, we'll have each node periodically send out "heartbeat" messages to one another over the network. A heartbeat message is a network message you use to figure out whether a remote node is still reachable; a heartbeat consists of a request ("Are you still online?") and an immediate response ("Yes, I am!"). By periodically sending heartbeat requests to all its peers and tracking which peers have not responded in a long while, a node can get a pretty good idea of which nodes are online and which aren't:

DIAGRAM

Next, we have each node

## Split-Brain

## A Temporary Setback

## Consensus

## Peer-to-Peer, Fault Tolerant Consensus

## Majority-Rules Voting















---

TODO

* Get to the point of showing split brain
* A section called "A Temporary Setback" in which we realize we were solving the wrong problem
* The new problem is getting a bunch of nodes to agree on something. 
* Solving this problem allows us to rescue single-leader-with-replication approach
* Solving this new problem would also solve the distributed variable problem without the single leader approach at all; we won't need a leader at all any more, we just allocate a copy of the variable on every node and run this algorithm to make the nodes agree what the latest value is.
* Now we're ready to define this as the consensus problem and list its properties
* We can consider revisiting single leader with failover as a broken consensus algorithm that's broken because it's not fault tolerant
* Segue into the idea of voting, that puts us back on the track we were on in the old draft with like half the word count

---

































---









---

<div markdown="1" style="width: fit-content; border: 1px solid #eee; padding-left:1em; padding-right: 1em">
**Table of Contents**

1. [Consensus Problems](#part1)
2. [Designing a Consensus Algorithm](#part2)
2. [FLP](#part3)
3. [Paxos](#part4)

</div>

Ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" and they'll probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos. 

These algorithms are several kinds of amazing. For one, they’re the bedrock on which the entire online world is built. Every Internet service you rely on has a consensus algorithm running in there somewhere. Few developers interact with these algorithms directly on a day to day basis, but all the databases and cloud services we build on are themselves built on consensus algorithms. Consensus is everywhere!

It’s kind of incredible that these algorithms exist at all. They do things that ought to be impossible. The environment of distributed systems is unforgiving; platforms provide surprisingly few guarantees, and the guarantees you get on paper don’t always hold in practice: hardware malfunctions, software crashes, networks glitch. It’s a world where anything that can go wrong will go wrong, is going wrong, and has been going wrong for weeks unnoticed. Working on distributed systems is like playing a perverse game of Simon Says, where you make assumptions you think are already very conservative, only to find out that &ndash; Simon didn’t say! &ndash; that assumption can break too. Look at the world that way, and it seems almost impossible that an algorithm running in that environment could ever provide complete and utter certainty, and still work pretty darn reliably. But that’s exactly what consensus algorithms manage to do! No wonder they’re popular.

On top of all that, it’s a small miracle that we managed to discover a working consensus algorithm at all. The first one was the culmination and many years of hard work by many intelligent people, and that process was a long journey, full of false starts and wrong directions. Along the way, many were certain an impossibility proof was just around the corner. In fact, even when the first working algorithm was published, people didn’t get it &mdash; people came out of the author’s presentation thinking it was some kind of practical joke! (It may not have helped that the author was wearing an Indiana Jones getup, presenting the algorithm as if it were some kind of archaeological find, but still!)

Today, the struggle to develop and understand consensus algorithms continues. One of the biggest recent advancements in this space was published in a paper titled *In Search of an Understandable Consensus Algorithm* ... and that’s 25 years after Paxos, the first working algorithm, was published! The original author of the Paxos algorithm published the it twice, but people still don’t get it. Much ink has been put to paper trying to explain how consensus algorithms work, to no avail. Many people have written blogs trying to make Paxos easy to understand, and failed. In this little book, I aim to repeat their folly!

The truth is, neither Raft nor Paxos will eever be all that easy to understand: the train of thought that leads you there is too long and too winding to be covered quickly. But people come away thinking these algorithms are things normal people will never be able to grasp, and that's a problem we can fix!

We're going to retrace the original line of thought that led to the discovery of Paxos, the first consensus algorithm discovered. Our discussion will be self-contained and complete: if you can write the backend for a modestly large web service, you have enough background to get through this thing. I won't use big words or mathematical notation when I don't have to, but I'm not going to go easy on you either &mdash; no handwaving, stretching metaphors or oversimplifying. At the end, you're going to understand the core Paxos algorithm, and you'll have a clear picture how somebody could have come up with it.

In chapter 1, we'll start by exploring the problem space. We'll nail down exactly what a consensus algorithm does, and when you’d want to use one. In chapter 2, we’ll start trying to design a consensus algorithm &mdash; only to run into a dead end! In chapter 3 we'll talk about FLP, a major discovery that makes it clear why the things we were doing in chapter 2 didn't work and points to a potential path forward. Finally, in chapter 4, we'll use what FLP taught us to fix our broken designs, and end up with Paxos &mdash; the first working consensus algorithm. It'll be a long, but rewarding journey. Buckle up!

<center>
  <a name="part1"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    1: Consensus Problems
  </h1>
</center>
---

TODO one big problem with this draft is we never really nail down what "shared state" means. We should probably change up the model a little bit to start with "multiple nodes with replicas of the same state" plus "the ability for multiple threads to update it." Or something like that. The point is in needing for the update to happen on all replicas or none so they stay in sync. Only once you've done that, can you start building other invariants, like an account only being created once, or primary key uniqueness.

Also, right at the beginning of chapter 2 I say the word "consensus variable," so either explicitly define that as a term of find a better term.

However, one difficulty here is the definition involving replication precludes the ability to explore the leader algorithm, which is an interesting discussion to have. We can "forget and then remember" the replication factor argument we set up in this chapter, but that's poor form

---

In the dictionary, the word "consensus" roughly means "agreement." In the world of distributed algorithms, the word consensus has a bit more connotation. Consensus algorithms resolve conflicts that would otherwise leave the system in an ambiguous, indeterminate state. Let's see what that means:

## A Real-Life Metaphor

A lot of computing terms come from real-life metaphors. The real-life metaphor behind consensus algorithms is getting a group to agree on something.

Think about a time you and a group of friends went to a restaurant. Obviously you must have picked some place to go; do you remember how you did that? Life's most important decisions are tough, but the decision where to go eat that day or night probably wasn't: I'm guessing you were looking for an option that was good enough for everyone, rather than the platonically ideal place a person could eat. Everyone already agreed they wanted to go eat somewhere; not knowing where to go was standing in the way, so you all picked a place nobody was *unhappy* with and off you went! This is the kind of "agreement" we're thinking of when we talk about consensus in the context of consensus algorithms. We want a quick conflict resolution, not necessarily an optimal decision.

The dictionary definition of “consensus” may basically be, “agreement,” but in our context we’ll use the word to mean “agreement that lets a group move forward.” Our definition of consensus implies making a decision just to resolve ambiguity and be able to make progress. And, for that reason, it also implies sticking to any decision previously made, lest we reintroduce the ambiguity that was standing in our way.

As distributed systems programmers, we often find this kind of ambiguity standing in our way too. Frequently we find ourselves updating some shard state from multiple threads running concurrently on different servers. If both thread updating some shared variable have to anticipate the other might be doing a conflicting update at the same time, we need some way to resolve the conflict so that both threads agree which update happened, just for both threads to be able to move on. Let's see how *that* works.

## Example: Account Signups

For this next part, I'm going to need an example website that allows people to sign up and pick a username. There are lots of websites that do this; some are fun websites like Reddit or Instagram, others are work websites like your bank or your managing a credit card. For the sake of discussion, I'll pick on Reddit; but feel free to sub in your own website of choice.

Now, Reddit is a pretty big service. A lot of people read and write posts on Reddit every day. It's too much data and too much load for any one server to handle; so we can be pretty sure Reddit is implemented as a distributed service running across many servers. I don't know about you, but personally I've never needed to worry about which server I'm connected to or how Reddit's servers themselves are connected to one another; I just point my browser to their URL and go look at pictures of cats. I can do that because Reddit presents itself as a single cohesive service, rather than as a pile of individual servers I move between. Services like these are rife with potential for conflicts.

Say Alice and Bob want to both create an account called `paxos_expert`. Both open the signup page, both see the username is available, and both hit register. At this point, Reddit has two account creation requests for the same username. Obviously, both users can't have the same name, so the service will have to decide who gets the username and who gets a "username already taken" error. But Alice and Bob might be connected to different servers; how will the two servers discover the conflict, resolve it, and present a single shared timeline of events that's consistent for both Alice, Bob, and everybody else on Reddit? That's a consensus problem.

Consensus problems appear all of the place in systems like these, because there are lots of servers interacting on shared state. Any time you have the following ingredients, you have a consensus problem on your hands:

* Shared state (like the user account table)
* Multiple servers acting on the shared state (Alice and Bob creating accounts)
* A conflict between updates (Alice and Bob can't both use the same username)
* The need for a globally consistent answer (everyone should be clear whose account it is)

Put these things together, and you're all but forced to find a quick, arbitrary, and permanent way to resolve the conflict globally. But you're not looking for an optimal answer; there is no optimal choice between giving the `paxos_expert` username to Alice vs Bob. Rather, the need for a decision is just to make sure all threads of the service are consistent with one another. We need for Alice and Bob to agree who got the account, so the servers have to agree who registers it.

Now, you may have implemented an account signup page before, and if so I'm guessing "distributed consensus" was nowhere on your radar. How is it that you can run into consensus problems all over the place, and yet never have used a consensus algorithm before? Consensus problems turn out to be very easy to delegate: you don't have to worry about consensus because you have a toolbox full of databases and cloud services that already solve consensus for you. But if you keep digging, the buck has to stop somewhere; eventually, if you pass down through all layers of delegation, you'll eventually find a 'true' consensus algorithm like Raft or Paxos. That's why people say these algorithms are "foundational" to the field.

So let's keep digging. How did Reddit (probably) solve the problem of two people registering the same account name at the same time? Probably with the help of a database. Databases have consensus problems too.

## Example 2: Database Replication

Most people interested in consensus algorithms are basically familiar with relational databases with tables, rows columns and primary keys, so I won't belabor the point here. What matters to our discussion is that, even though we've never seen Reddit's source code, we can be pretty sure the accounts are stored in a database, using some kind of uniqueness constraint (such as choice of primary key) to prevent two users from registering the same username.

At the scale of something like Reddit, even the database of accounts is probably too much data and too much load for any one server to handle; so it's likely the database is broken up into pieces (sometimes called **partitions** or **shards**) which each served separately. And, to ensure that a single server crash or disk failure doesn't bring down the account database, most likely there are backup copies (called **replicas**) of each database partition. Figuring out how to accept database updates that affect multiple partitions and apply the updates to all replicas is a difficult problem, core to modern database design, and there's a whole zoo of solutions used in practical systems today. Many of those problems are consensus problems.

For example, one strategy sometimes used to ensure greater uptime and provide more write throughput is to allow every replica of a database or database partition to accept updates locally, and lazily synchronize the updates between the different replicas. This technique is called **multi-leader replication**. Say for the sake of discussion that Reddit stores accounts in a database that uses multi-leader replication.

So, Alice and Bob both show up and both try to register the username `paxos_expert`. That itself is a consensus problem, but the web servers don't resolve that problem amongst each other; instead, each forwards a database request to insert a row with the username column set to `paxos_expert`, and let the database figure out. If the database uses a multi-leader strategy, and those two servers happen to connect to different replica servers, then the database now has two replica servers with conflicting row insert operations; another consensus problem:

[[ diagram showing the two web servers delegating to two replica servers ]]

In this diagram, you can also see how the consensus problem has been delegated from the web servers down to the database, by transforming the duplicate usernames problem at the web tier to a conflicting row insert problem at the database tier. This is why consensus algorithms are very important to people who build foundational primitives, services and databases for the cloud, but are largely invisible to people who encounter consensus problems on a day-to-day basis.

## Example 3: Lock Service

As a final example, let's consider a distributed lock service. These things implement the networked version of the mutex locks you may have encountered in multithreaded code. A thread which obtains a distributed lock can be certain no other thread on any server also holds the lock at the same time. Locks are a pretty low-level primitive with quite a few applications. For example, say we implement an in-memory account for recently accessed Reddit accounts; we might use a distributed lock service to decide which nodes cache which accounts. That way, if a node crashes, it loses its lock; later, some other cache server takes the lock and begins caching those accounts, thereby allowing the caching system to recover from the server crash.

Say we have two account cache servers, Alpha and Bravo, which both see an account is not yet cached and both try to take a lock on the account and cache it. By definition of a lock, they can't both take the lock at the same time, so the lock service needs to guarantee one server gets the lock, one doesn't, and both agree who has the lock. As with all our examples, it doesn't really matter whether Alpha or Bravo gets the lock; it's only important that both servers are crystal clear which server got the lock.

## Consensus Problems: Recap

We have seen three examples where servers can get into disagreement with one another. In general, this seems to happen when you have multiple servers, shared state, and two updates to that state happening on two different servers at the same time. If those two updates aren’t compatible with one another, the servers end up in conflict, and that conflict must be resolved before they can move on and finish processing the user’s request. Since they need agreement to move forward, we call these conflicts *consensus problems*. Algorithms we can use to resolve these conflicts are accordingly called *consensus algorithms*. 

We also began to see how these kinds of conflicts are inherent to the kinds of systems we’re building. A distributed service is mode of servers on a network, but users don’t have to worry about the network or servers; they treat all of it as one  cohesive service and interact with the service as if it were running on one computer. When the user expects there to be one state (the service’s state), but the state is actually replicated across many servers, conflicting updates are basically inevitable. Resolving these conflicts requires consensus algorithms, which perhaps is why people consider consensus to be foundational algorithms for the field of distributed systems.

> Yet consensus algorithms are not inevitable. In some special cases, there's another way to deal with concurrent updates without resolving conflicts: just design the system so that no two updates can ever conflict! This is the idea behind a class of data structures called [conflict-free replicated data types](https://en.m.wikipedia.org/wiki/Conflict-free_replicated_data_type) or “CRDTs.” Although these things are interesting and have received renewed interest of late, designing systems on top of traditional consensus algorithms is tried, true and usually simpler to reason about.

Finally, we also saw how easy it was to delegate consensus problems to lower-level primitives; so even though consensus problems are everywhere, consensus algorithms are usually only ever found at bottom-level, foundational services. Even so, someone has to actually solve the problem eventually. Soon, that someone will be us.

## Fault-Tolerant Consensus

So far, consensus algorithms might not sound very hard to design. There are lots of simple heuristics for resolving conflicts in a consistent way: for example, all nodes could pick the update that has the earliest timestamp, or have some way to sort the nodes and pick the update that came from the node with the lowest ID. Sadly, the only reason consensus sounds simple so far is because we’re missing an entire dimension of the problem. Let's talk about reliability for a second.

Think about the device you're using to read this book; have you ever had weird little problems with it? Freezes, or crashing apps, weird glitches, system-wide slowdowns, overheating, unexplained network disconnects, blue screens, anything like that? I’m guessing these things have happened to you, even if they don't happen often enough to be a major disruption day to day.

But now imagine you were using not just one your device, but a thousand of them. Or imagine you’re a cloud provider, running hundreds of thousands of these things across the world, connected by thousands of miles of network cables. How often do you think you'd be dealing with these kinds of little problems? Heck, you'd probably never be able to fully rid yourself of them, no matter how hard you tried! Rare problems, multiplied by thousands of machines, become common. If you serve a million requests every minute, you should expect every possible "one-in-a-million" problem to happen once every minute! (That's over 1,400 times a day). There’s no way to squash all of these problems, so code running on this environment &mdash; and, by extension, you &mdash; will have to live with them. These little problems are called **faults**, and software that works despite these faults is said to be **fault-tolerant**.

Consensus algorithms that can be deployed in practical, real-world settings need to be fault-tolerant. Designing fault-tolerant consensus algorithms turns out to be *hellishly* difficult. These algorithms need to provide exacting guarantees, but run on an imperfect platform that itself fails to provide its stated guarantees routinely. How does one do that? It’s something we’ll have to tackle soon.

## Properties of a Fault-Tolerant Consensus Algorithm

So far, we know that a consensus problem is an update conflict (or the potential for a conflict) between two servers which each want to update the same state. While we're still exploring the problem, let's get crisper on the problem statement: what guarantees exactly does a consensus algorithm need to provide to be useful in practice?

People have already tackled this question in The Literature&#8482; using Big Words&#8482;. Those big words are **agreement**, **validity**, **termination** and **fault tolerance**. To make sure we're on the same page as the rest of the field, we'll use those words too.

### Agreement

Let's think about the lock service for a minute. What makes a lock useful is the mutual exclusion guarantee: the fact that one server has the lock allows it to assume no other server has the lock. If this guarantee cannot be enforced, the lock is useless! Let's think about a couple ways this could go wrong in a distributed setting.

What if two servers disagreed which of the two has acquired the lock? There are two very bad things that could happen: first, you have the situation where each server believes it has the lock; then the mutual exclusion guarantee of the lock is violated, as it's no longer correct that one server holding the lock guarantees no other server has it too. Second, you have the situation where both servers believe the other one got the lock; then both servers will wait for each other forever, a situation called **deadlock**. We must not allow either of these to happen: a useful consensus algorithm must guarantee all servers agree which server got the lock.

Another problem: what if a server was allowed to 'change its mind' about who has the lock? Maybe Alpha initially thinks Bravo got the lock, but then later changes its mind and decides it does have the lock after all. Once again, the mutual exclusion guarantee would be violated: *initially* Bravo was correct in assuming it was the sole owner of the lock, but that assumption was violated the moment Alpha changed its mind. A useful consensus algorithm must not allow changing minds; once a decision is made, it must be committed forever.

These ideas are all encompassed in the idea of the **Agreement** property, which roughly means, "the algorithm only ever decides on one value." That applies across servers (all servers choose the same value) and across history (a server never changes its decision). Another way to say the same thing: a consensus algorithm presented with conflicting updates chooses which update to accept **at most once**.

### Validity

There's a very easy, pretty cheater way to implement the Agreement property: hardcode a single answer. For example, if you make a "distributed lock service" which does nothing except hardcode the return value, "Node Alpha has the lock," technically that algorithm provides agreement &mdash; even though it's completely useless a lock service. A real lock service should always grant the lock to some node currently requesting it. Similarly, a consensus algorithm should always one pick one of the updates somebody actually proposed; we'll call this the **Validity** property.

This might seem obvious, but it pairs with the Agreement requirement in an interesting way. It's easy to pick a deterministic 'decision rule' for selecting a value; for example, "hash all the values and pick the one whose hash has the lowest numerical value." But by Validity, a consensus algorithm doesn't start knowing what the candidates are, which means the process of discovering candidates runs concurrently with making a decision. According to the Agreement rule, the system must not change its mind if it already chose one value and later discovered a value which is 'better' according to the decision rule.

### Termination *

Before you had a consensus algorithm, you had a conflict that needed resolving. What good would a consensus algorithm be if it runs forever, without resolving the conflict? Whether or not you used that algorithm, you'd still have a conflict and it'd still not be resolved. We should probably require that the algorithm stops and produces a decision at that point; we require **Termination**.

Another way of defining Termination is to say the algorithm produces a decision **at least once**. Recall the Agreement property can be defined as defining a decision at most once; so Agreement and Validity together imply the need to make a decision **exactly once** per run of the algorithm. (However, this will prove to be problematic later on; hence the asterisk in the title heading. More on this at the end of chapter 3.)

### Fault Tolerance

As we discussed before, in a real distributed systems, there are little hardware and software problems happening all the time; things like:

* Hardware failures
* Server crashes
* Power loss
* Servers taken down for upgrade
* Network disconnects
* Slow networks
* Lost network messages
* Network messages delivered multiple times

These things are called **faults**; being able to work despite faults is called **Fault Tolerance**. A consensus algorithm that runs in the presence of faults must be fault tolerant, or it simply is not useful at all; in a large enough network, the odds of none of these things ever happening is astronomically low.

What exactly does it mean for a consensus algorithm to be Fault Tolerant? At a basic level, Agreement and Validity are non-negotiable; no matter what bad things happen in the platform the algorithm runs on, we must never allow servers to disagree or decide something nobody asked for; an algorithm that *can* do that is useless. However, the relationship between Fault Tolerance and Termination is a little wishy-washy. Even though you're the star coder I know you are, you cannot write code that is guaranteed to make a decision despite a "fault" where the power is cut to every server simultaneously. If all the computers are off, they ain't running your code. But we can ask for guaranteed termination in spite of a "reasonable" number of faults, even if the exact number is negotiable.

By the way, throughout this book, when we evaluate an algorithm’s fault tolerance, we always consider the absolute worst possible case. For example, if we want to claim an algorithm can tolerate one server crash, then it must be true that *any* server can crash without halting the algorithm. If, for example, most servers were allowed to crash but there was one ‘special’ server that the algorithm cannot allow to crash, then we’d technically say the algorithm does not even tolerate a single crash, because in the worst case, that special server could be the one crashed. This might seem harsh, but keep in mind that even rare situations &mdash; like that one special server crashing &mdash; become common at high scale.

> ### Consensus Properties: Recap
>
> **Agreement**: The algorithm only ever decides on one value
>
> **Validity**: The algorithm decides on a value some node proposed
>
> **Termination (*)**: The algorithm eventually decides on a value
>
> **Fault Tolerance**: The above are upheld even if the underlying platform faults
>
> * Agreement and Validity are upheld despite *any* number of faults
> * Termination is upheld despite *some* number of faults

<center>
  <a name="part2"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    2: Designing a Consensus Algorithm 
  </h1>
</center>

Just as the best way to understand a piece of code is to change it, the best way to understand an algorithm is to try designing it yourself. In this chapter, we’ll try to design a fault-tolerant consensus algorithm. We won’t be able to get all the way there in one chapter, but we’ll learn a few important things along the way.

## A Couple Restrictions

This discussion will get complicated fast. To keep things under control, we'll start by placing a few limitations on our design. The goal is to make it easier to get a basic working design to begin with, while leaving the door open to extending the basic design to support all the features we wanted. 

### Binary Decisions

For now, our algorithm will only allow two values to be proposed. To make those two options easier to talk about, we’ll give them names: the <span style="color:red">red</span> option and the <span style="color:blue">blue</span> blue option.

[diagram introducing the two colored circles we’ll use throughout the book]

Remember, these options can stand in for anything else: 0 and 1, yes and no, apples and oranges, etc. 

In real lilfe, it'd be useful to support any number of different candidate values. If we only support two candidates, then e.g. the lock service can only be implemented for two nodes, which isn't very useful. However, in most of computing, you end always being able to have exactly zero of something, exactly one of something, or $N$ of something; so if we can figure out how to have exactly two candidate values, there's probably an easy enough way to extend it from 2 to $N$ candidates.

Note that having two *values* doesn't restrict us to having only two *nodes*. We can have any number of nodes proposing values to the system, as long as each of those nodes always either proposes <span style="color:red">red</span> or <span style="color:blue">blue</span>.

### One-Shot Decisions

For now, let's have an algorithm that can only reach one decision and never change it. In terms of the lock service, this would mean one node acquires the lock and can never release it, which again is probably unrealistic. However, in most of computing, instantiating things is easy; so if we can figure out how to do a one-shot decision, maybe we can implement a stream of decisions by chaining a sequence of one-shot decisions. 

So, in conclusion: we'll start in this chapter designing a consensus algorithm for one-shot binary decisions: up to two candidate values (<span style="color:red">red</span> and <span style="color:blue">blue</span>) can be proposed, and the algorithm will decide on either value and then exit. 

## A Programming Interface

To nail down the design further, let's think about what kind of API we want give to clients using our consensus algorithm. The way I see it, we need two methods: one which inputs a candidate value into the system (for Validity) and one which outputs the decided-upon value (for Agreement). Let's go with these:

* **propose()**: offers the consensus system a value you'd like to change the state to, which can either be “red” or “blue”
* **query()**: ask the consensus system what value the state actually was changed to

Code that wants to update the shared state first calls **propose** with the value it prefers, followed by **query** to see what value was chosen. Code that just wants to read the shared state can skip **propose** and just call **query**. The consensus system guarantees every **query** call returns the same value, per the Agreement rule, and that the value returned is the value some node proposed, per the Validity rule.

If this is a good interface, we should be able to use it to solve consensus problems we already know of. We could make the lock service work by defining a shared consensus variable called `owner`, which is defined as:

* the ID of the node which currently holds the lock
* `null` if nobody has the lock yet

Initially, `owner` is null on every node. When a node wants to acquire the lock, it proposes its own ID for `owner`, and then queries `owner` to see whether it got the lock, like this:

```
// returns true if this node got the lock, false otherwise
try acquire lock() {
  consensus.propose(owner=me)
  return consensus.query(owner) == me
}
```

If two nodes call this simultaneously, only one node's proposal will be selected, and both nodes' **query** calls will return that node's ID. The node with that ID will then return true to the calling thread, and do whatever it needs to do under the lock; the other node returns false to the caller, indicating it does not have the lock and should do something else.

Make sense? Then let’s take our first stab at a design:

## A Leader-Based Algorithm

Given a network of computers, we pick one of the nodes in the network and designate it the **leader**. Maybe a human sets a config option on all nodes in the network so they all know the network address of the leader. We then make the leader the center of all activity: it receives all proposals, decides which one to accept, and returns the accepted proposal on queries.

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

The leader isn't very complex either. It just remembers the first proposal received by any client, and returns that proposal upon request:

```
leader {
  value: Proposal; // the accepted proposal, red or blue
  
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

* **Agreement**: ✅ &mdash; every node always returns whatever value the leader stores, and the leader never changes what it stores after the first proposal
* **Validity**: ✅ &mdash; the leader only stores a proposal that somebody sent it
* **Termination**: ✅ &mdash; a decision is reached after only one message is sent
* **Fault Tolerance**: hmm ... we have a problem here.

It would seem the leader is a **single point of failure** &mdash; if it crashes, or loses power, or gets disconnected from the network, or any number of other bad things happen to the leader, nobody else is going to be able propose or query the consensus variable. Even one fault halts the entire algorithm; so the algorithm is not fault-tolerant.

But maybe we can rescue this design? What if we maintain replicas of the leader’s state, and promote one of the replicas to leader when the leader fails? It’s a good idea, save for one major problem: that only helps if you already have a working consensus algorithm to begin with.

## Failover is a Consensus Problem

Promoting a backup node to leader only works if all nodes in the network agree that backup node is now the leader. If some nodes think one node is the leader and other nodes think another node is the leader, there are two leaders in the system. If the two leaders get out of sync, then the two sets of nodes listening to the two leaders start returning different values from their **query()** method, and Agreement is violated. 

Consider the following network. Node 1 is currently the leader; nodes 2-5 are following. Nobody has made a proposal yet, so the current consensus `value` on the leader is `null`:

[diagram]

Now let's say the network faults, splitting the network: now nodes 1-2 can talk to each other, but not to nodes 3-5; likewise, nodes 3-5 can talk to each other, but not nodes 1-2. Our system has now split into two sub-networks:

[diagram]

This situation, where groups of nodes can talk to other nodes within a group but not the rest of the network, is called a **network partition**. It can happen in practice, for example, if a network switch which connected the two groups of nodes got unplugged or something.

Well, here's the problem: node 1 was the leader, and nodes 3-5 cannot talk to it; for all they know node 1 might have crashed. So they fail over to a new leader. Let's say they pick node 3:

[daigram]

Uh oh, now there are two leaders! And now that there are two leaders, they can decide on different things. Say node 2 proposes <span style="color:blue">blue</span> and node 4 proposes <span style="color:red">red</span>. Then the different sub-networks end up with different accepted proposals:

[diagram]

Now a client that queries the system can either receive <span style="color:blue">blue</span> or <span style="color:red">red</span>; clearly the Agreement property is violated. This situation, where a system expects to have one leader but actually ends up with two or more, is called **split-brain**. To prevent split-brain, we need a way to get all nodes to agree which node is currently the leader node.

Alas, getting all nodes to agree which node is currently the leader is itself a consensus problem. If some nodes believe that leader is still reachable, they believe the leader should not be changed; if other nodes can't contact the leader, they believe the system needs to fail over to a new leader. That's a conflict between nodes. We must resolve that conflict &mdash; get every node to agree on a leader &mdash; before we can proceed with the rest of our consensus algorithm. But resolving the conflict requires use of a consensus algorithm we have not yet built. So we can't rely on leader failovers at all until we have a working consensus algorithm first.

We did learn an important lesson: consensus problems are very good at hiding in plain sight. Maybe it has something to do with how easy it is to delegate a consensus problem to some other subsystem. Plenty of ideas which look like potential solutions to consensus turn out to themselves rely on a solution to the consensus problem. So if we ever find an easy answer, we should first check if the solution is easy only because it assumes an existing solution of the very consensus problem we’re trying to solve.

So where does that leave us? The starting example I chose didn't work out, but it does suggest a new direction: we need to design a **peer-to-peer** algorithm. Instead of putting one node in charge, we need to set things up so nodes cooperate as equals, haggling out which proposal should be accepted.

Do you know of any way to do that?

## Second Stab at a Design

I'll bet you’ve used peer-to-peer consensus algorithms in real life:

Imagine the restaurant example once again. What exactly happens once you as a group realize you all need to decide on a place to go? Maybe someone throws out an idea, someone else throws out a different one, someone agrees, someone disagrees. Before you know it, a group opinion has formed. Boom, consensus reached!

Kinda sounds like voting to me ... do you think we could make an algorithm out of voting?

## The Majority-Rules Voting Algorithm

We can totally code up something that throws out proposals and votes on them, just like in the restaurant example above. But unlike real life, where people have preferences, we’ll code an algorithm where each node has zero preference and just votes for whichever option it heard about first, and never changes its mind. Here's a sketch:

Every node will have its own local accepted `value`, which is the value it heard about first and voted for. At the start of the algorithm, every `value` is null. Each time a client of our system calls **propose()**, proposing red or blue, we send that proposal to all nodes, including ourselves. When a node receives its very first proposal, it updates its `value` variable to the porposed value, thereby "voting" for it, and then sends out its own round of message to all its peers. After that, whenever a node receives a proposal, it ignores it because it has already voted.

Then we implement **query()** by tallying votes: ask every node what it voted for (what its current `value` variable holds), and see if any proposal has been voted in by a majority (more than half of the nodes). If so, that is the consensus decision; otherwise, the system is considered to still be deciding; the client gets an error saying to wait a little bit and retry.

Here's the same design sketch again, as pseudocode:

```
consensus {
  value: Proposal; // the accepted proposal
  nodes: Node[]; // all nodes in the network
  
  init {
    value := null // no accepted proposal yet
  }
  
  // this is the propose() API
  propose(proposal) {
    // nodes.all includes this node as well
    nodes.all.send(proposal)
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
    counts: map{ from proposal to int }
    foreach node in nodes {
      counts.add(node.get_current_value())
    }
    
    return get_majority_proposal(counts) 
  }
  
  on value requested(peer) {
    peer.reply(value)
  }
}
```

The sketch is almost complete. One problem we still need to deal with is split votes.

What if every node votes, but no proposal reaches a majority. For right now we only allow the proposal to be one of two values, red or blue, but even so, if we have an even number of nodes we can end up with a perfect tie: half of the nodes vote red and half vote blue. If that happens, we run out of votes and exit, but have not yet made a decision; that violates the Termination rule ("at least one decision is made").

It's okay though, we can work around that problem relatively easily: just require that the algorithm run on an odd number of nodes. That way, if every node has voted, the two vote counts cannot be equal, so one of either red or blue must have gotten more votes. That's the value we decide on.

Okay, so ... does *this* design work?

* **Agreement**: ✅ &mdash; only one proposal can reach a majority, and that's the proposal **query()** always returns; so at most one value can be returned by **query()**
* **Validity**: ✅ &mdash; nodes only vote for a value someone proposed; so the value that got the most votes was proposed by somebody
* **Termination**: ✅ &mdash; a decision is reached after all votes are in
* **Fault Tolerance**: . . . 😀 I told you this would be tricky, didn't I? 

No, this algorithm isn't fault-tolerant either! But we're getting better at this &mdash; this time, the problem is much more subtle.

## Faults and Split Votes

This design is pretty resilient, but depending on how things play out, it sometimes has a **window of vulnerability** where one *very* poorly timed crash could deadlock the algorithm This case might be rare, but it still happens as a result of just one crash, so by worst-case analysis we technically have to admit this algorithm cannot withstand just one crash &mdash; hence, it's not fault-tolerant.

More specifically, the problem is that split vote again. We can fix split votes by having an odd number of nodes, but if just one node crashes, we're back to having an even number of nodes, reintroducing the possibility of a tie. Let's watch this in action:

Say we have a cluster of 7 nodes. One proposes <span style="color:blue">blue</span>, the other <span style="color:red">red</span>:

[diagram]

Off to the races! Those two nodes each send their proposals, <span style="color:blue">blue</span> and <span style="color:red">red</span>, to all their peers. Remember, each of those peers will vote for whichever proposal it receives first, then never change its mind. Since we have two proposals propagating at the same time, the choice will come down to small timing variances. Let's say the race continues for a while, and just happens to turn out neck-and-neck, three votes each for <span style="color:blue">blue</span> and <span style="color:red">red</span>. Just one node is still decided because it hasn't received a proposal yet:

[diagram]

What happens if that undecided node crashes right now?

[diagram]

Now we're in trouble. A proposal needs 4 votes to reach majority and be accepted; but the voting is over, and no proposal has 4 votes! Seems like we're stuck.

Can we get the algorithm un-stuck from this point? Maybe! Earlier on we said we couldn't use simple deterministic "tiebreaking" rules as a consensus algorithm because a consensus algorithm must discover new candidate values concurrently with making a decision, and must ensure a decision once made never gets changed even if a 'better' candidate is discovered later. But in this case, all the voting is done, so all the discovering is done. So maybe we can include some kind of tiebreak rule? For example, we could say, "in the event of a tie vote, red always wins" (chosen fairly by asking my 4 year old what his favorite color is).

It's alluring, because it *almost* works. *Almost*!

Right now, all we know is the undecided node is currently unreachable; we actually don't know whether it's offline or just running slowly:

[diagram with 3 red, 3 blue, one thought bubble with a question mark inside]

It could be that the undecided node has actually crashed and not coming back. In this case, "3 votes for red, 3 votes for blue" is the final state of the system, so having a tiebreak rule like "in the event of a tie, assume red has won" is a great idea and allows us to decide in spite of the split vote and inopportune crash:

[diagram with 3 red, 3 blue, one empty node crossed out to indicate its gone. Scribble in "interpretation: red wins"]

But, it's just as possible the undecided node is simply running slowly. In that case, the 3v3 split is *not* the final state of the system, and having a tiebreak rule would be a disaster: you would be able to transition directly from:

[copy and paste the diagram above, which still says "interpretation: red wins"]

to:

[diagram with undecided node now decided blue, with "interpretation: blue wins"]

Whoops, the algorithm changed its mind! We absolutely cannot allow that. So we can't have a tiebreak. But if we can't have a tiebreak, then we can't fix the split vote situation. So we can't make the majority algorithm decide in spite of even one crash. This whole majority voting thing was so promising at the beginning, and now it's starting to look like a dead end.

Dang.

<center>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Recap &amp; Intermission
  </h1>
</center>

We've covered a lot of ground. Let's reflect on where we've been before we move forward.

We started our journey by examining consensus problems: situations where multiple nodes want to update the same state at the same time. If these updates conflict, the nodes must find a way to resolve the conflict with one another, or else neither node has a way to make progress.

We came up with four basic requirements any consensus algorithm should have:

* **Agreement**: The algorithm only ever decides on one value

* **Validity**: The algorithm decides on a value some node proposed

* **Termination (*)**: The algorithm eventually decides on a value

* **Fault Tolerance**: The above are upheld even if the underlying platform faults

  * Agreement and Validity are upheld despite *any* number of faults

  * Termination is upheld despite *some* number of faults

We saw that consensus problems are everywhere, and we saw how consensus problems can be delegated from component to component, allowing us to rely on a few foundational consensus algorithms to provide these four guarantees in a wide variety of different primitives, freeing us from having to deal with consensus algorithms ourselves.

And, we also saw that coming up with a working consensus algorithm is surprisingly tricky.

We started with a leader-based algorithm, but realized the leader is a single point of failure, leading to an algorithm which is not fault tolerant. We tried to come up with some kind of failover system where a follower node which is up-to-date can be promoted to leader, but then we realized to do this while avoiding split brain, we'd need an external solution to the very consensus problem we were trying to solve in the first place. A dead end.

Then we examined majority voting, which was a pretty good analogy to how this problem is actually solved by people in the real world. But we ran into a problem with split votes: we could avoid them by having an odd number of nodes, but even one node crashing can lead to an even number of nodes, resurrecting the possibility of a split vote. We also looked at tiebreaking schemes, but realized running the tiebreak prematurely could cause the algorithm to change its mind in the case no node crashes, which is unacceptable. Another dead end.

Something interesting I'd like to point out right now: we have now hit two dead ends, and the stories are strangely similar. Each time, we first came up with an algorithm that works when no node crashes, but found the algorithm can deadlock if even one node crashes. And each time, we then tried to find something we could do to 'fix' the deadlock once we were stuck (like failing over to a new leader, or coming up with a tie-break), only to find the fix could violate Agreement in the case *no* nodes crash. Isn't it weird how that happened twice?

We're now at the low point of the story; we started with high hopes, but we ran into many setbacks, and now we have nothing to show for all our hard work. In the next chapter, we will begin our ascent: we'll figure out exactly what's wrong, move past the problem and ultimately end up with a working solution.

But first, I have homework for you. This is just a book; I can't make you do it. But if you take me up on this right now, you'll end up with a much better feel for what comes next.

This is the point in the thought process where the field as a whole got stuck for many years. I'd like to invite you to get stuck for a while too. Your assignment is to step away from your computer and think over everything we've talked about.

* What went wrong with our two approaches?
* Can you think of a way to fix either one?
  * Or an entirely different approach that works?
* What's up with the same dead end appearing two different ways?

If you take me up on this exercise, three things to keep in mind:

1. If you think you have the solution, remember to do an absolute-worst case analysis: like with the voting example above, if there's one specific execution where the loss of one specific node at the exact wrong time deadlocks the algorithm, it's not fault tolerant. Sometimes, finding that one situation is tricky!
2. Be wary of solutions which appear to work, but actually rely on some other form of consensus. For example, in our single-leader algorithm, we found that getting all nodes to agree on a leader is itself a consensus problem, and thus we couldn't rely on safe leader election to design a consensus algorithm in the first place.
3. If you find an algorithm that deadlocks, but you find a way to "fix" it so a decision is made despite the deadlock, double-check there's no way that decision can be made in the case where no node crashes and there is no deadlock; otherwise you might be permitting more than one decision, like with leader failover or vote tie-breaking

And most importantly, have a little fun! Mess around, try things, be interested by setbacks instead of getting frustrated. This is a game, not a test. Nobody's watching. Richard Feynman famously liked to "play" with physics; now I'm asking you to play with consensus algorithms.

And if that really isn't your thing, you can just read on.

<center>
  <a name="part3"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    3: FLP
  </h1>
</center>

## When the Going Gets Tough, the Tough Prove the Going is Tough ... and Give Up

Before a working consensus algorithm was discovered, people chewed through this problem just as you might have during the intermission. And they kept running into the same dead end. They could make an algorithm the provided Agreement, Validity and Termination in the case where no node crashes, but Fault Tolerance was elusive. There was always an annoying little window of vulnerability, a case where a single crash would be enough to deadlock the system. Attempts to deal with deadlock directly were futile; they would always end up violating Agreement one way or another. A choice between Agreement and Fault Tolerance is no choice at all; we must have both.

Some advice: when you're trying to solve a problem, and you keep hitting the same dead end no matter what you do, the next thing you should try is to prove impossibility: maybe you can show the dead end will *always* come up because of some aspect of the problem space, and thus prove the problem actually isn't solvable. Not trying to solve the unsolvable would certainly save you some time!

Well, that's exactly what three researchers did in the mid-1980s. In their paper *Impossibility of Distributed Consensus with One Faulty Process*, Fischer, Lynch and Paterson (the "FLP" in "FLP result") explained exactly why nobody could come up with an always-fault-tolerant consensus algorithm. Their paper uses the language of formal mathematics to explain their ideas in the form of a proof, but tied up in all the notation and state machines and network models is a set of simple and insightful idea. In this chapter, we'll see what they saw.

We're going to do this thing *in medias res* style. First we'll see *exactly* what you can't do in a fault-tolerant consensus algorithm, then we'll catch up to how they ever thought of such a weird thing, and finally we'll figure out how it means we need to change our strategy. Without further ado, here it is:

## The Weird Thing You Cannot Do (Doctors Hate It!)

The FLP result gives us a piece of extremely valuable advice: we must never design a consensus algorithm which can get into the following situation:

> There is some network message $m$ with the following properties:
> 
> * $m$ can be delivered *before* a decision has been reached
> * In all cases, *after* $m$ has been delivered and processed, the system is guaranteed to have reached a decision

If such a message can appear at any point in your algorithm, you won’t be able to make the algorithm tolerate a single fault.

I know it looks like legalese, but I'm saving you from pages of technical definitions and mathematical notation here, so bear with me. Read the above *carefully* and make sure you understand what's being stated before moving on.

Ready? Alright, if you're like me, this might raise a handful of questions, like:

* Do both of our example algorithms actually do that? (They do.)
* Why does something so specific consistently make it into our designs without us noticing? (It has to do with Termination.)
* What exactly goes wrong? (It's the same ‘dead end’ we've seen twice already.)
* How did they come up with this? (I assume it took them a long time.)

Let's explore each of these questions in depth.

## Do both our algorithms really do that?

They do.



## How do we keep accidentally creating this situation?

It has to do with Termination.



## What exactly goes wrong?

It’s the same dead end we’ve seen twice now.

## How did they come up with this?

TODO the idea of a decision point which is driven by the relative order of two message deliveries. That the “has always decided once processed” thing is a concise and neutral way to say, either this message makes the decision or find a the decision was already made and backs off (the atomic decide-if-not-decided). That a message sent after the decision is made doesn’t affect the ability to remain bivalent.







---

Let’s try a flow like this:

1. What doesn’t work is (message from lemma 3, stated in the paper’s precise yet somewhat obtuse manner)
2. What the heck does that even mean? Do a majority voting 3-node example and point out the votes that do that
3. Why does this case show up? Preview the idea of an algorithm that terminates has a finite number of messages. If you’re going to stop sending messages, one of those messages had better be guaranteed to make the decision, or you exit with no decision 
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



