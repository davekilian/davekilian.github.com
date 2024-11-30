---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

Distributed consensus algorithms are a critical piece of modern computing infrastructure that almost nobody really understands. The very first one, *Paxos*, was initially met with met with indifference; people thought it was some kind of elaborate joke. (It probably did not help the presenter was dressed up in an Indiana Jones costume.) For the next 25 years, there was really only one vaible algorithm for distributed consensus, and that algorithm was Paxos. For systems built in the past decade, there has also been a second choice: *Raft*. Raft exists because Paxos is too difficult; in fact, the Raft paper is called *In Search of an Understandable Consensus Algorithm*. However, Raft is also fundamentally a lot like Paxos; it is better factored and the mechanics are a little easier to explain, but the fundamental approach to solving consensus is basically the same (and just as weird-looking) as Paxos.

Not everyone thinks Paxos is impossible to understand. Its inventor for one claims that Paxos is "among the simplest and most obvious of distributed algorithms," and that it "follows unavoidably from the properties we want it to satisfy." Although it's fun to joke this is because Dr. Lamport exists in some astral plane that we mortals can only hope to one day glimpse, the fact remains that he's right: the confusing part about Paxos isn't Paxos, it's understanding the problem space and the solution space well enough to see why you end up needing to solve consensus exactly the way Paxos does it.

In this post, we're going to try it that way. First we're going to discuss the problem of consensus, to see why it matters and also understand the mechanics of what a consensus algorithm needs to do. Then we're going to try out several "obvious" algorithms that see that they don't work. In trying to figure out why they don't work, we'll end up discussing the *FLP result*, which uses the language of formal proves to explain what is wrong with the obvious approaches we tried so far, and eliminates entire classes of things that seem like they should work but don't. When we redouble our efforts using what the FLP result tells us, we'll stumble right into Paxos &mdash; or, more specifically, the core "synod" algorithm that underlies the algorithm.

# Part 1: Consensus

One of the fun (or maybe "fun") parts of programming distributed systems is that trivial things you normally do in normal code turn out to be impossible, or boderline impossible in a distributed system. One is a problem I will call **exactly once**: how do you write code to make something happen exactly one time?

## Example: Exactly-Once

For example, say we have a function called `thingy()` and we want to make sure the program calls `thingy()` exactly one time. This is incredibly simple to do in regular, single-threaded code: you just call `thingy()`:

```java
thingy();
```

In multithreaded code, it might be a bit trickier: maybe we have a bunch of threads that all *can* call `thingy()`, but we want to make sure only one thread actually does it. We could accomplish that by putting `thingy()` behind a lock:

```java
synchronized (thingyLock) {
  if (!thingyAlreadyCalled) {
    thingy();
    thingyAlreadyCalled = true;
  }
}
```

The boilerplate is unfortunate, but otherwise the solution is simple.

What about in a distrubted system? If we had some kind of 'distributed lock' primitive, maybe we could reuse the solution above, but for now let's assume we have no fancy abstractions or tools, just the ability to send and receive messages on the network. Then how do we ensure only one machine calls `thingy()`?

Here's one idea: for most distributed systems running in data centers or in the cloud, we know ahead of time the full set of nodes that are going to participate in the distributed system. We can provision each node with a list of all nodes in the system, and then run an algorithm over that list to choose which node is the one that gets to call `thingy()`. One simple way to do this is the so-called *bully algorithm*, which works on the principle "the biggest guy wins:" pick some sort attribute of each node (maybe an integer "node ID") and then choose the node with either the largest or smallest value in that sort order. Example:

```java
int bullyAlgorithm(Collection<int> nodeIds) {
  return Collections.max(nodeIds); // min() would work too!
}
```

This algorithm does not require nodes to send or receive any network messages to decide which node gets to be the one to call `thingy()`; instead, we give each node the same node list, and run the same determinstic algorithm over that list, which causes each node to independently come to the same conclusion as to which node should be the `thingy()` caller. 

With this, we can now solve the exactly-once problem for distributed systems, even ones which use multithreading:

````java
boolean iAmTheLeader = bullyAlgorithm(nodeList);
if (iAmTheLeader) {
  synchronized (thingyLock) {
    if (!thingyAlreadyCalled) {
      thingy();
      thingyAlreadyCalled = true;
    }
  }
}
````

The code has gotten even more complex, and provisioning that node list is a little bit of an operational burden, but overall this seems pretty tractable! Let's move on to another related problem, which I'll call the **happened-or-not?** problem.

## Another Example: Happened-or-Not?

A situation that comes up a lot in online services is having two users try to do two incompatible things at the same time; since the actions are in conflict with one another, your code must resolve the conflict by accepting one and rejecting the other.

For example, maybe you have an online ordering system where an order, once placed, can be cancelled up to a certain point, but then becomes uncancelable later on when you ship the item. What do you do if someone tries to cancel their order at the same time the warehouse is marking the order uncancelable so they can ship the item? How do we know if the cancellation happened or not?

Once again this is simple enough to solve in normal single-threaded code: we just need a variable with three states, "happened," "did not `happen`," and "still undecided." For order cancellation, those states might be called "cancelled," "shipped" and "pending" respectively:

```java
enum OrderState {
  PENDING, // cancellable
  SHIPPED, // no longer cancellable
  CANCELLED, // no longer shippable
}
```

We can use an instance of this enum to ensure a shipped order cannot be cancelled and vice versa:

```java
OrderState orderState = OrderState.PENDING;

boolean tryCancel() {
  if (orderState == OrderState.PENDING) {
    orderState = OrderState.CANCELLED;
    return true;
  } else {
    return false;
  }
}

boolean tryShip() {
  if (orderState == OrderState.PENDING) {
    orderState = OrderState.SHIPPED;
    return true;
  } else {
    return false;
  }
}
```

Once again, we can extend this into a multithreaded solution by adding a lock:

```java
boolean tryCancel() {
  synchronized (orderStateLock) {
    if (orderState == OrderState.PENDING) {
      orderState = OrderState.CANCELLED;
      return true;
    } else {
      return false;
    }
  }
}

boolean tryShip() {
  synchronized (orderStateLock) {
    if (orderState == OrderState.PENDING) {
      orderState = OrderState.SHIPPED;
      return true;
    } else {
      return false;
    }
  }
}
```

We can also once again distribute the algorithm by using a bully algorithm to elect a coordinator. However, we need to send network messages: `tryShip()` and `tryCancel()` will both be RPCs to the leader node selected by the bully algorithm. We can actually use the exact implementation above as the server side of the RPC, and implement the client side by just forwarding requests to the server like this:

```java
boolean tryCancel(Collection<int> nodeIds) {
  int leaderId = bullyAlgorithm(nodeIds);
  return rpc(leaderId, "tryCancel");
}

boolean tryShip(Collection<int> nodeIds) {
  int leaderId = bullyAlgorithm(nodeIds);
  return rpc(leaderId, "tryShip");
}
```

At first glance, the exactly-once and happened-or-not problems don't seem all that related: their single-threaded solutions are completely different. However, it's interesting that we could extend the respective single-threaded solutions to multithreded and distributed solutions using the same basic techniques: in both cases, we could multithread by putting the single-threaded solution behind a lock, and we could distribute by electing single coordinator node using a bully algorithm. That must mean these two problems have *something* in common, even if the solutions are totally different. We'll come back to that a little later. For now, I want to talk about something missing from our distributed solutions for these two problems: they're not fault tolerant.

## Fault Tolerance

Distributed algorithms are a lot like multithreaded algorithms. After all, every distributed algorithm already has multiple threads: there are multiple machines, and each machine has at least one thread. So all problems that exist in multithreaded algorithms also exist in distributed algorithms.

The new dimension of problems distributing adds on top of multithreading is dealing with **faults** in the system: hardware can lose power, software can crash, and networks can degrade or become disconnected. Technically these problems also affect single-node systems, but single-node code generally doesn't care about these faults because, if one occurs, then the node has failed and the code is no longer running. In other words, these faults do cause a mess, but nobody expects single-node code to be able to deal with that mess because the computer running the code is down. In a distributed system, a fault can affect a subset of the system's nodes while leaving code running on the remaining nodes to deal with the mess.

Both our distributed solutions to the exactly-once and happened-or-not problems rely on a "coordinator" node to make progress, and neither can deal with a fault bringing the coordinator down: the bully algorithm will continue to pick that node as the coordinator, even though it's not online and can't do its work. In the exactly-once solution, the coordinator is the only node allowed to call `thingy()`, so if the coordinator crashes we never call `thingy()` and thus fail in our ultimate goal to call `thingy()` exactly once. In happened-or-not, the coordinator is the only node that stores the current cancellation state, so if it crashes, other nodes will either fail or hang in their `tryShip` / `tryCancel` calls. Neither of our distributed solutions so far is **fault tolerant**.

The lack of fault tolerance is certainly a limitation, but it may not actually be a problem! If you're operating a small network of nodes, in a highly controlled environment such as a data center, and you have the option to regularly schedule "maintenance windows" for taking the whole system offline and doing upgrades, then non-fault-tolerant distributed algorithms may work for you. As we have already seen, non-fault-tolerant distributed algorithms are often pretty simple, much more so than any of the algorithms we're going to go on and build in the rest of this article. However, at a certain network size it becomes infeasible to ensure all nodes are always online, and in many Internet-facing services it is not feasible to take the system down for maintenance, ever. In these situations, the only option is to design software that can tolerate faults that affect a subset of the system. Consensus algorithms are usually employed by people trying to make fault-tolerant systems, so for the rest of this article, we will assume fault tolerance is a non-negotaible requirement.

It turns out to be very difficult to come up with fault-tolerant solutions to the exactly-once and happened-or-not problems. However, there is a nifty way to factor out the fault tolerance part of the problems, such that we build one fault-tolerant primitive and use it to solve these completely different problems (and many others). That fault-tolerant primitive is called a **consensus algorithm**.

## Consensus

"Consensus" means "agreement," usually in the context of a group decision. In distributed systems, consensus is the problem of getting all nodes in a distributed system to agree upon the value of some variable.

At its heart, a consensus algorithm is an asynchronous conflict-resolution algorithm: any number of threads input a 'propsed' value they each would like to set the variable to, the consensus algorithm picks one of the proposed value arbitrarily, and then all threads stick the value the algorithm chose. Now all threads are in agreement!

The interface for using a consensus algorithm might look something like this:

```java
class Consensus<T> {
  /** Input a proposed value, eventually learn the winning value */
  public Future<T> resolve(T proposal);
  
  /** Learn the winning value without proposing one */
  public Future<T> get();
}
```

Code that has an opinion of what value the algorithm should pick calls `resolve()`, passing in the proposed value it wants the algorithm to pick, and waits on the future to wait for the algorithm to finish executing. When the future resolves, it returns the value the consensus algorithm picked. Code that just wants to learn what value is picked without proposing anything calls `get()`.

For now, let’s not worry about how to implement such a thing; let’s assume we have a fault-tolerant distributed algorithm implementing the interface above, figure out how to use it to solve our example problems.

TODO walk through happened-or-not. This one first because the integration is downright trivial.

TODO walk through exactly-once. This one's a bit tricky, if `thingy()` has side effects then you have cross-domain transaction sort of problems. If we assume `thingy()` is a pure function and we just want the result saved, then the strategy is for everyone to call `thingy` and all try to resolve it to the return value, so only one invocation (arbitrarily) takes effect.

TODO hints lots of other things are easy to build on consensus algorithms. There's a reason these are considered foundational to modern distributed systems.

## Properties of a Consensus Algorithm

TODO exploring how the algorithm is used in the examples above, derive agreement, validity, termination

## Implementing Consensus

TODO redo the same basic ladder as with the other two examples:

* There is a variable
* Single-threaded solution sets the variable, resolving the future for everyone immediately
* Multithreaded solution sets the variable under a lock, resolving the future for everyone immediately
* Distributed (not fault tolerant) hops over to the bully leader and does the multithreaded solution there
* Fault tolerant consensus?
  * First it seems easy. Then upon closer examination, it seems impossible. Luckily, there's a subtle workaround that makes it not quite impossible.
* Segue out into section 2

---

TODO

Part 2 is called Voting Algorithms

* Motivate one-shot red-vs-blue decision and how we should be able to generalize it later
* A real-life fault-tolerant consensus algorithnm is majority rules voting
* Explain the basic algorithm 
* Split votes
* Workaround to have an odd number of nodes fails if we lose a node
* Tiebreakers
* Workaround to use a tiebreaker can fail if no nodes fail!

Part 3 is FLP, I had several thoughts of how to explain this without getting into the formal math

* At a high level, the core claim they make is you have three bad choices
  * Algorithm doesn't guarantee a decision is ever made => stupid, it doesn't terminate
  * Algorithm has one way it can force a decision => can't be fault tolerant, that step may never execute
  * Algorithm has 2+ ways to force a decision => it's broken in the case where no node fails
* Most of the work in the proof is proving the 2+ case
  * Intuition: you can't differentiate "never" from "not yet" i.e. "failed" vs "slow"
  * Their system model precludes the existence of timeouts, but that is correct, we don't want to rely on brittle timeouts
  * Kicked off theory around "what if I had a failure detector" => great, but IRL we don't have this failure detector
* Apply this back into the language of the majority voting example
  * Idea of a "one-way" vs "two-way" decision
  * The bad case is "two-way"
  * An algorithm which never makes a two-way vote never terminates, stupid
  * Our initial algorithm had one two-way decision, but it wasn't fault tolerant (split votes)
  * Adding a tiebreak added two two-way decisions, and introduced a race condition
  * All of this is exactly as FLP predicted
* Are we stuck? No, there's a suble workaround, we don't actually need termination as long as probabilities converge
  * Doesn't matter if we don't guarantee termination if it becomes vanishingly unlikely to keep not terminating
  * Quote the FLP paper which I believe direclty suggests this, but doesn't manage to find the algorithm to do it
    * Eh probably they may also have thought this approach would not pan out?
* So what does a majority voting algorithm that never forces a decision to be made? Let's examine one. It's called Paxos

Part 4 is Paxos

* Key ideas
  * Voting progresses in rounds, potentially infinitely many rounds
  * In each round, the vote is one-sided: either we choose this specific candidate, or we remain undecided



