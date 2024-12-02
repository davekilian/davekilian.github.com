---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

TODO a couple things to check / correct here

* Paxos was first submitted in '89 but not published until '98
* Viewtamped replication may predate it?
* I'm not sure it's fair to say Raft is just a refactoring of Paxos actually
  * Leader election is just majority voting with timeout randomization to avoid split votes, that's like Paxos
  * What about recovery of replicated log?
  * Wikipedia's explanation sounds like you can lose quorum-committed entries if the new leader happened to never get that entry, which is clearly wrong / the explanation seems to be incomplete. Will have to check the paper
  * Figure 7 looks related

---

Distributed consensus algorithms are a critical piece of modern computing infrastructure, but few people really understand how they work. When the first consensus algorithm, *Paxos*, was introduced to the world, it was met with vague confusion: people in the first presentation suspected they were being pulled into some kind of elaborate joke. (It probably did not help that Dr. Lamport, inventor and presentor, gave the talk dressed up as Indiana Jones.) For the next 25 years, Paxos remained the only viable algorithm for distributed consensus, and during that time it gained such notoriety for being impossible to understand that when a second viable consensus algorithm, *Raft*, finally came along, the paper was called *In Search of an Understandable Consensus Algorithm*. 

Here's the thing about Raft, though: at its core, it's pretty similar to Paxos. It is better factored and much easier to describe, but the fundamental way it goes about solving the consensus problem is the same as Paxos. Dr. Lamport, who invented Paxos, once wrote that Paxos is "among the simplest and most obvious of distributed algorithms," and that it "follows unavoidably from the properties we want it to satisfy." The fundamental similarity between Raft and Paxos seems to support this. But if Paxos is so simple and falls directly out of the problem definition, why is it so hard to explain? Why do direct explanations and metaphors like the 'part-time parliament' alike fail to make the algorithm seem intuitive?

I think the answer is that Paxos seems straightfoward to people who already know the problem space: what we're trying to do, what obvious approaches don't end up working, *why* they don't end up working, and how you work around that. So let's try it that way: let's define the consensus problem in context of real situations where we would use a consensus algorithm, then let's try the obvious approaches that don't work, let's see why they don't work, and then hopefully we will be lead to Paxos "unavoidably, from the properties we want it to satisfy."

# Part 1: Consensus

Just about every distributed system has a consensus algorithm running it in somewhere; even if your code doesn't do it, the dadtabases you're using probably do, and the cloud services you deploy your code on *definitely* do. But what makes consensus so fundamental? Why does it pop up everywhere?

One of the fun (or maybe "fun") parts of programming distributed systems is that trivial things you normally do in normal code turn out to be really hard, or maybe even impossible in distributed code. Consensus helps turn problems like that back into something tractable. To see what I mean, let's try a couple of examples:

## Example: Picking a Random User

TODO I think this is going to fall apart as soon as I try to change this to consensus, because we want to do one-shot consensus initially, and 

---

Llet's say we're writing code for a message board website, and we want to add a 'user of the day' function where we spotlight one particular user at random each day. How do write code to pick a user once, and only once each day?

In normal code, this is pretty easy to do: just write code that picks a user:

```java
User getUserOfTheDay() {
  if (!this.lastUpdated.equals(Date.today())) {
    this.userOfTheDay = User.randomUser();
    this.lastUpdated = Date.today();
  }
  
  return this.userOfTheDay;
}
```

What if we need to call this method from multithreaded code? Easy enough, just put it all behind a lock:

```java
User getUserOfTheDay() {
  synchronized (this.lock) {
    if (!this.lastUpdated.equals(Date.today())) {
      this.userOfTheDay = User.randomUser();
      this.lastUpdated = Date.today();
    }

    return this.userOfTheDay;
  }
}
```

Let's ramp up the difficulty. What if we have distributed system?

In a way, a distributed system is lot like a multithreaded system, just with the different threads running on different computers. For us as programmers, what this changes in practice is how threads communicate: on a single machine threads can just share variables, but once we have different threads on different machines we have to use the network. Our multithreaded solution relies on sharing variables (while holding a lock), so to adapt it to a distributed environment, we will need to use the network instead. But otherwise it doesn't need to change all that much.

Here's a simple approach: we'll select one node to be responsible for the user of the day. Let's call that node the **coordinator**. The coordinator is the only node that stores the current user of the day, and is solely responsible for selecting new users each day; all other nodes contact the coordinator when they need to know today's user of the day.

It's a good idea, but we have a bootstrapping problem: how do all our nodes figure out who the coordinator is? Well, for most distributed systems running in data centers or the cloud, we know ahead of time the full set of nodes that are going to participate in the distributed system. If we provision each node with that node list, we could have all nodes independently run the same deterministic algorithm on that list to pick the coordinator. As long as all nodes have the same node list and run the same deterministic algorithm, they'll all end up picking the same coordinator &mdash; without ever sending a single network message!

A simple algorithm we could run on the node list to select a coordinator is the so-called *bully algorithm*, which works on the principle, "the biggest guy wins." We pick some attribute on which to sort the node list, and then choose the node that is largest or smallest in the resulting sort order:

````java
int bullyAlgorithm(Collection<int> nodeIds) {
  return Collections.max(nodeIds);
  // ... Collections.min() would work too!
}
````

Once the coordinator is known, all nodes in the system can do a remote method call to the coordinator node to ask it for the current user of the day, or to ask it to update the current user of the day:

```java
void getUserOfTheDay() {
  int coordinatorNodeId = bullyAlgorithm(App.nodeIds);
  remoteCall(coordinatorNodeId, "getUserOfTheDay");
}
```

On startup, if a node runs the bully algorithm and realizes it is the coordinator, it starts the server side of this remote method call interface, implementing it using the same implementaiton we've already seen a few times:

```java
if (bullyAlgorithm(App.nodeIds) == App.myNodeId()) {
  registerRemoteCall("getUserOfTheDay", () => {
    synchronized (this.lock) {
      if (!this.lastUpdated.equals(Date.today())) {
        this.userOfTheDay = User.randomUser();
        this.lastUpdated = Date.today();
      }

      return this.userOfTheDay;
    }
  });
}
```

Phew! That took a lot of thinking, but fortuitiously we ended up with a small enough snippet of code to fit into a blog post. Problem tackled; let's try another.

## Example: Order Cancellation

In networked server code with many users, it's not uncommon for two users to try to do incompatible things at the same time. How do we make sure we accept one action and reject the other? For example, let's say we have an online ordering system where an order, once placed, can be cancelled up until it is shipped from the warehouse. What if someone in the warehouse tries to mark the order as shipped (no longer cancellable) at exactly the same time the customer tries to cancel the order?

Once again this is simple enough to solve in normal single-threaded code: we just need a variable with three states: order pending (still shippable and cancellable), cancelled (no longer shippable), and shipped (no longer cancellable):

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

Looks good to me. How about we multithread this? Once again, putting the key pieces behind locks seems to work:

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

How about turning this into a distributed solution? We can do it the same way we did before: expose the implementation above as an RPC call from some coordinator node, and have all other nodes call into the coordinator when they need to deal with the order state. The client side might look like this:

```java
OrderState getOrderState() {
  int coordinatorNodeId = bullyAlgorithm(App.nodeIds);
  return remoteCall(coordinatorNodeId, "getOrderState");
}

boolean tryCancel() {
  int coordinatorNodeId = bullyAlgorithm(App.nodeIds);
  remoteCall(coordinatorNodeId, "tryCancel");
}

boolean tryShip() {
  int coordinatorNodeId = bullyAlgorithm(App.nodeIds);
  remoteCall(coordinatorNodeId, "tryShip");
}
```

\And the server might look like this:

```java
if (bullyAlgorithm(App.nodeIds) == App.myNodeId()) {
  registerRemoteCall("getOrderState", () => {
    synchronized (orderStateLock) {
      return orderState;
    }
  });
  registerRemoteCall("tryCancel", () => {
    synchronized (orderStateLock) {
      if (orderState == OrderState.PENDING) {
        orderState = OrderState.CANCELLED;
      }
    }
  });
  registerRemoteCall("tryShip", () => {
    synchronized (orderStateLock) {
      if (orderState == OrderState.PENDING) {
        orderState = OrderState.SHIPPED;
      }
    }
  });
}
```

It's interesting that this problem could be multithreaded and distributed the same way as the user of the day example, even though the single-threaded implementations have nothing in common. Maybe there's some way we could factor out the multithreading and distributed systems parts of our solutions to our two example problems. Here's how:

## A Refactoring: Write-Once Variables

Say we have a primitive I'll call a **write-once variable**. The interface looks something like this::

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable haa been initialized */
  public Future<T> get();
}
```

This thing can be implemented using the exact same pattern we used for our two example problems. For single-thread code, we just implement the interface:

```java
class WriteOnce<T> {
  private CompletableFuture<T> value = new CompletableFuture<>();
  private boolean initialized = false;
  
  public void tryInitialize(T value) {
    if (!initialized) {
      value.complete(value);
      initialized = true;
    }
  }
  
  public Future<T> get() {
    return value;
  }
}
```

To multithread, we just add a lock:

```java
class WriteOnce<T> {
  private Object lock = new Object();
  private CompletableFuture<T> value = new CompletableFuture<>();
  private boolean initialized = false;
  
  public void tryInitialize(T value) {
    synchronized (lock) {
      if (!initialized) {
        value.complete(value);
        initialized = true;
      }
    }
  }
  
  public Future<T> get() {
    return value;
  }
}
```

To make it distributed, TODO do I really want to try to show this? urgh, the future aspect makes it all boilerplatey.

## A Curveball: Fault Tolerance





---

TODO continue refactoring: 

* Point out so far everything has seemed quite achievable. This is because we are missing the fault tolerance aspect
* Now thanks to this refactor step we have only one algorithm to make fault-tolerant, which is the write once variable. Intro that making a write-once variable fault tolerant is called the consensus problem, and explain how "agreement" relates to write-once semantics since that's really not clear at all.
* Properties of a consensus algorithm from analysis of our example problems, how they rely on consensus.
* 

---



## Fault Tolerance

Distributed algorithms are a lot like multithreaded algorithms. After all, every distributed algorithm already has multiple threads: there are multiple machines, and each machine has at least one thread. So all problems that exist in multithreaded algorithms also exist in distributed algorithms.

The new dimension of problems distributing adds on top of multithreading is dealing with **faults** in the system: hardware can lose power, software can crash, and networks can degrade or become disconnected. Technically these problems also affect single-node systems, but single-node code generally doesn't care about these faults because, if one occurs, then the node has failed and the code is no longer running. In other words, these faults do cause a mess, but nobody expects single-node code to be able to deal with that mess because the computer running the code is down. In a distributed system, a fault can affect a subset of the system's nodes while leaving code running on the remaining nodes to deal with the mess.

Both our distributed solutions to the exactly-once and happened-or-not problems rely on a "coordinator" node to make progress, and neither can deal with a fault bringing the coordinator down: the bully algorithm will continue to pick that node as the coordinator, even though it's not online and can't do its work. In the exactly-once solution, the coordinator is the only node allowed to call `thingy()`, so if the coordinator crashes we never call `thingy()` and thus fail in our ultimate goal to call `thingy()` exactly once. In happened-or-not, the coordinator is the only node that stores the current cancellation state, so if it crashes, other nodes will either fail or hang in their `tryShip` / `tryCancel` calls. Neither of our distributed solutions so far is **fault tolerant**.

The lack of fault tolerance is certainly a limitation, but it may not actually be a problem! If you're operating a small network of nodes, in a highly controlled environment such as a data center, and you have the option to regularly schedule "maintenance windows" for taking the whole system offline and doing upgrades, then non-fault-tolerant distributed algorithms may work for you. As we have already seen, non-fault-tolerant distributed algorithms are often pretty simple, much more so than any of the algorithms we're going to go on and build in the rest of this article. However, at a certain network size it becomes infeasible to ensure all nodes are always online, and in many Internet-facing services it is not feasible to take the system down for maintenance, ever. In these situations, the only option is to design software that can tolerate faults that affect a subset of the system. Consensus algorithms are usually employed by people trying to make fault-tolerant systems, so for the rest of this article, we will assume fault tolerance is a non-negotaible requirement.

It turns out to be very difficult to come up with fault-tolerant solutions to the exactly-once and happened-or-not problems. However, there is a nifty way to factor out the fault tolerance part of the problems, such that we build one fault-tolerant primitive and use it to solve these completely different problems (and many others). That fault-tolerant primitive is called a **consensus algorithm**.

## Consensus

"Consensus" means "agreement," usually in the context of a group decision. In distributed systems, consensus is the problem of getting all nodes in a distributed system to agree upon the value of some variable.

In practice, a consensus algorithm implements something like a write-once variable. The variable starts off uninitialized and therefore cannot be read. The first thread to try to write the value succeeds, and the variable is initialized ot the value that thread was trying to write. Afterwards, any attempt to write the value again silently fails; the variable stays set to the value that was first set. Once the variable has been written the first time, it can be read by any thread anywhere in the system, at any time. Assuming the algorithm is distributed and fault tolerant, all of these functions keep working even if some nodes of the network have failed.

The interface for using a consensus algorithm might look something like this:

```java
interface Consensus<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable haa been initialized */
  public Future<T> get();
}
```

Initially the variable stored by a `Consensus` object is not initialized, so any thread calling `get()` will receive a future that has not yet resolved. The first thread to call `tryInitialize()` writes its proposed value to the variable and resolves all futures previously returned by `get()` to the value that was just written. Subsequent calls to `get()` return a resolved future, and subsequent `tryInitialize` calls do nothing, because the variable is already initialized.

So if  what we have here a variable, why do we call this a "consensus" algorithm? The name *consensus* comes from the tricky part of this algorithm: how do we deal with the case where the variable is not yet initialized, and *multiple* threads try to initialize it at the same time? If that happens we must arbitrarily pick one of those `tryInitialize()` calls to accept and discard the rest. Since all nodes must *agree* which `tryInitialize` call is the one that succeeded, we call the algorithm *consensus*. In other words, the core of any distributed consensus algorithm is a mechanism for resolving a conflict and coming to agreement.

So, assuming we have a fully distributed, fault-tolerant implementation of this write-once variable, how do we use it to solve our example problems?

### Exactly-Once

TODO walk through exactly-once. This one's a bit tricky, if `thingy()` has side effects then you have cross-domain transaction sort of problems. If we assume `thingy()` is a pure function and we just want the result saved, then the strategy is for everyone to call `thingy` and all try to resolve it to the return value, so only one invocation (arbitrarily) takes effect. Use this as a chance to talk about how exactly once cannot really exist in a distributed system, the best approximation available is to make it happen lots of times but only take effect once.

One way to help here is remove “thingy” and replace it with a function that is obviously pure, such as picking a random‘user of the day.’ That’s also nice because each example has a fundamental problem and a concrete example where it crops up.

### Happend-or-Not

TODO walk through happened-or-not. This one also has a few complications, we have to use it only to decide whether cancelled or shipped but not pending. We can kind of hack around it by assuming uninitialied variable / never written to means the order is still pending.

### ... and more!

TODO hints lots of other problems can be solved using consensus algorithms for distributed fault tolerance. There's a reason these are considered foundational to modern distributed systems.

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

Part 5 is about turning this into a write-many variable / state machine replication. We don't have to delve into the details, we basically just say

1. Replicate a log where each entry is individually a write-once consensus variable
2. Replaying the log deterministically rebuilds the state of any data structure
3. Paxos includes further optimizations for this situation (multi-paxos)
4. The way it does this is somewhat complex, and somewhat better factored in Raft

