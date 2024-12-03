---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

Distributed consensus algorithms are a critical piece of modern computing infrastructure, but few people really understand how they work. When the first consensus algorithm, *Paxos*, was introduced to the world in 1989, it was met with a sort of [indifferent confusion](https://www.microsoft.com/en-us/research/publication/part-time-parliament/), and it wasn't until 9 years later that it had gained enough grassroots popularity to be published in a major journal. As the world moved online and everyone started building their critical infrastructure as distributed systems, Paxos found its way into the foundation of almost everyone of those newly built systems. But, at the same time, it also grew notorious for being beyond the comprehension of mere mortals &mdash; so much so that, when a second viable consensus algorithm, *Raft*, finally came along well over a decade later, the paper was called *In Search of an Understandable Consensus Algorithm*.

I like Raft, and I think if you're choosing between Raft and multi-Paxos for a project today, Raft is very like to be the better choice. Multi-Paxos is presented only as a rough sketch in the original Paxos paper, and although a hodgepodge of other papers try to fill in the blanks, they aren't completely consistent with one another, so it's up to you to partially reinvent the wheel. At least with Raft, there's a complete algorithm spelled out in one place! Even so, I don't think Paxos should be skipped over entirely. It's true the full multi-Paxos algorithm is complex and under-specified, but the core of Paxos (a little protocol called the *synod algorithm*) is small, well-specified, insightful, and can be embedded into other systems and algorithms in a way that the full Raft and multi-Paxos cannot. (TODO: Find an example. I thought Aurora does some kind of row-level Paxos using the synod algorithm? Am I misremembering?)

Leslie Lamport, who invented Paxos, [once described Paxos's core synod algorithm](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) as "among the simplest and most obvious of distributed algorithms," and claimed that it "follows unavoidably from the properties we want it to satisfy." He's right, although it might not seem that way on a first read-through of the algorithm. Paxos's core algorithm falls out of a lot of invisible context: you need to really understand the problem we are solving, why obvious solutions fail, and what must be done to work around those failures, for the synod algorithm to start looking like a clear and minimal solution. So that's what we'll do: let's take a fresh look at the problem of consensus, try to solve it, get stuck, and then get un-stuck; we should end up reinventing the synod algorithm, which is the core of Paxos. Along the way we'll cover the FLP result, which bridges the gap from obvious algorithms that don't work, and algorithms (like Paxos) that do.

# Part 1: Consensus

Just about every distributed system has a consensus algorithm running it in somewhere. Even if you have never written code to use a consensus algorithm directly, the fancy distributed replicated database you're using itself relies on a consensus algorithm, as do any cloud services you might be using. But what makes consensus so fundamental? Why are they ubiquitous in distributed systems?

One of the fun (or maybe "fun") parts of programming distributed systems is that quite a few common tasks that are trivial in normal code turn out to be really, really hard in distributed code. Consensus algorithms help turn problems like that back into something more tractable. To see what I mean, let's lay a couple of examples where a consensus algorithm might be helpful:

## Example: Picking a Random User

Let's say we're writing code for a message board website, and we want to add a 'user of the day' function where we spotlight one particular user at random each day. How do write code to pick a user once, and only once each day?

In normal code, the solution is not too complex. It might look like this:

```java
User getUserOfTheDay() {
  if (!this.lastUpdated.equals(Date.today())) {
    this.userOfTheDay = User.randomUser();
    this.lastUpdated = Date.today();
  }
  
  return this.userOfTheDay;
}
```

What if we need to call this method from multithreaded code? Easy enough, to make it thread-safe just put all the logic behind a lock:

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

What if we have distributed system? In a way, a distributed system is lot like multithreading, just with the different threads running on different computers. So hopefully we should be able to reuse most of our multithreaded solution above.

For us as programmers, what's different about distributed systems is how threads communicate: on a single machine, threads can just share variables, but once we have different threads on different machines we need to use the network. Our multithreaded solution relies on sharing variables, so we'll have to use the network now that we're distributed.

Here's an approach: we can assign to one node the job of managing the user of the day. Let's call that node the **coordinator**. The coordinator decides who is the user of the day, and remembers that decision all day. Any time some other node needs to know who is the current user of the day, it asks the coordinator.

This'll work, but now we have a bootstrapping problem: how do all nodes figure out which one is the coordinator? Well, for most distributed systems running in data centers or the cloud, we know ahead of time the full set of nodes that are going to participate in the system. If we provision each node with a copy of that node list, we could have all nodes independently run the same deterministic algorithm on that list to pick the coordinator. As long as all nodes run the same deterministic algorithm with the same node list as input, they'll all end up picking the same coordinator &mdash; without ever sending a single network message!

As to what algorithm we should run on the node list, a common choice is the so-called *bully algorithm*, which works on the principle, "the biggest guy wins." We pick some attribute on which to sort the nodes in the node list, and then choose the node that is largest or smallest in the resulting sort order:

````java
int bullyAlgorithm(Collection<int> nodeIds) {
  return Collections.max(nodeIds);
  // ... Collections.min() would work too!
}
````

Once the coordinator is known, all nodes in the system can do a remote method call to the coordinator node to ask it for the current user of the day:

```java
User getUserOfTheDay() {
  int coordinatorId = bullyAlgorithm(App.nodeIds);
  return remoteCall(coordinatorId, "getUserOfTheDay");
}
```

On startup, if a node runs the bully algorithm and realizes it itself is the coordinator, it starts the server side of this remote method call interface, which it implements the same way we did for multithreading:

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

Phew! That took a lot of thinking, but fortuitiously we ended up with a small enough snippet of code to fit into a blog post. Handwaving away all the gnarly networking code sure helped with keeping the implementation simple. But anyways, problem tackled! Let's try another.

## Example: Order Cancellation

In server code, lots of tings are happening all at the same time; sometimes two users try to do two conflicting things at the same time. How do we make sure we accept one user's action and reject the other? For example, let's say we have an online ordering system where an order, once placed, can be cancelled up until it is shipped from the warehouse. What if someone in the warehouse tries to mark the order as shipped (no longer cancellable) at exactly the same time the customer tries to cancel the order?

Once again this is simple enough to solve in regular, run-of-the-mill single-threaded code. We just need a variable with three states, order pending (not yet shipped, can still be cancelled), cancelled (no longer can be shipped), and shipped (no longer can be cancelled):

```java
enum OrderState {
  PENDING, // not yet shipped, can be cancelled
  SHIPPED, // no longer cancellable
  CANCELLED, // no longer shippable
}
```

We can use an instance of this enum to ensure a shipped order cannot be cancelled and vice versa:

```java
OrderState orderState = OrderState.PENDING;

void tryCancel() {
  if (orderState == OrderState.PENDING) {
    orderState = OrderState.CANCELLED;
  }
}

void tryShip() {
  if (orderState == OrderState.PENDING) {
    orderState = OrderState.SHIPPED;
  }
}
```

Looks good to me. How about we multithread this? Once again, putting the key pieces behind locks works pretty well:

```java
void tryCancel() {
  synchronized (orderStateLock) {
    if (orderState == OrderState.PENDING) {
      orderState = OrderState.CANCELLED;
    }
  }
}

void tryShip() {
  synchronized (orderStateLock) {
    if (orderState == OrderState.PENDING) {
      orderState = OrderState.SHIPPED;
    }
  }
}
```

How about turning this into a distributed solution? We can do it the same way we did before: expose the implementation above as an RPC call from some coordinator node, and have all other nodes call into the coordinator when they need to get or updatethe order state. The client side might look like this:

```java
OrderState getOrderState() {
  int coordinatorId = bullyAlgorithm(App.nodeIds);
  return remoteCall(coordinatorId, "getOrderState");
}

void tryCancel() {
  int coordinatorId = bullyAlgorithm(App.nodeIds);
  remoteCall(coordinatorId, "tryCancel");
}

void tryShip() {
  int coordinatorId = bullyAlgorithm(App.nodeIds);
  remoteCall(coordinatorId, "tryShip");
}
```

And the server might look like this:

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

It's interesting that we have two different problems, both with completely different single-threaded solutions, which could nonetheless be multithreaded and then distributed using the exact same approaches. In fact, there is an opportunity for refactoring here: we can isolate the code that does the multithreaded / distributed parts of these solutions by introducing a new abstraction, which I'll call a *write-once variable*.

## Write-Once Variables

Say we have a primitive I'll call a **write-once variable**. The interface looks something like this::

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable haa been initialized */
  public Future<T> finalValue();
}
```

The way we implement this thing should look pretty similar by now. First we write a single-threaded implementation:

```java
class BasicWriteOnce<T> implements WriteOnce<T> {
  private CompletableFuture<T> value = new CompletableFuture<>();
  private boolean initialized = false;
  
  public void tryInitialize(T value) {
    if (!initialized) {
      value.complete(value);
      initialized = true;
    }
  }
  
  public Future<T> finalValue() {
    return value;
  }
}
```

To make it thread-safe for multithreading, we just add a lock:

```java
class MultithreadedWriteOnce<T> implements WriteOnce<T> {
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
  
  public Future<T> finalValue() {
    return value;
  }
}
```

Then to make it distributed, we store the variable on a coordinator and have all other threads contact the coordinator for each operation. The full distributed solution, including client and server, might look like this:

```java
class DistributedWriteOnce<T> implements WriteOnce<T> {
  private int coordinatorId;
  
  private int bullyAlgorithm(Collection<int> nodeIds) {
    return Collections.max(nodeIds);
  }
  
  public DistributedWriteOnce() {
    coordinatorId = bullyAlgorithm(App.nodeIds());
    if (coordinatorId == App.myNodeId()) {
      runCoordinator();
    }
  }
  
  public void tryInitialize(T value) {
    return remoteCall(coordinatorId, "tryInitialize");
  }
  
  public Future<T> finalValue() {
    return remoteCall(coordinatorId, "finalValue");
  }
  
  private void runCoordinator() {
    WriteOnce<T> impl = new MultithreadedWriteOnce<>();
    
    registerRemoteCall("tryInitialize", (value) => {
      impl.tryInitialize(value);
    });

    registerRemoteCall("finalValue", () => {
      AsyncResponse<T> response = new AsyncResponse<>();
      impl.get().thenAccept(val => response.respond(val));
      return response;
    });
  }
}
```

Once again we were able to multithread the initial solution with a lock and distribute the multithreaded solution by selecting a coordinator node. Now let's see how this lets us *remove* the lock and the coordinator from our other two problem solutions by rewriting them on top of the `WriteOnce<T>` abstractions.

## Refactoring with Write-Once Variables

Let's start with the random user-of-the-day problem. The core question behind that one was how to make something happen once; in this case, namely, how to make sure one, and only one random user is picked each day.

Here's a way to do that with a write-once variable:

```java
User getUserOfTheDay() {
  User randomUser = User.randomUser();
  
  WriteOnce<User> userOfTheDay = // ...
  userOfTheDay.tryInitialize(randomUser);
  return userOfTheDay.finalValue().get();
}
```

The idea is to have every thread generate a random user of the day, but only allow the first such generated user to become today's user of the day; all other `tryInitialize()` calls silently have no effect because the write-once variable has already been picked. (To complete the solution, we would need a way to get a new `WriteOnce<User>` each day, but that's not important for this discussion so I'm going to handwave it.)

Do you see how the multithreaded / distributed guarantees of `WriteOnce<T>` are automatically conferred onto this implementation of `getUserOfTheDay()`, without us having to do anything special? If we pass in a thread-safe `WriteOnce`, that makes `getUserOfTheDay()` thread-safe, and if we pass in a distributed `WriteOnce` then `getUserOfTheDay` is also automatically distributed. Pretty nifty, huh?

Now let's try the order cancellation problem. For that one, the core question was how to determine a winner if two users try to take conflicting actions at the same time. One possible solution is to have all attempts to ship or cancel an order race to be the first to call `tryInitialize` on a write-once variable; whichver is processed first is the one we accept, and all subsequent attempts are rejected automatically:

```java
WriteOnce<OrderState> orderResult = // ...

void tryCancel() {
  orderResult.tryInitialize(OrderState.CANCELLED);
}

void tryShip() {
  orderResult.tryInitialize(OrderState.SHIPPED);
}

OrderState getOrderState() {
  Future<OrderResult> resultFuture = orderResult.finalValue();
  if (resultFuture.isDone()) {
    return resultFuture.get();
  } else {
    return OrderState.PENDING;
  }
}
```

Once again, the internal guarantees of thread safety / distribution provided by the `WriteOnce<OrderState>` confer thread safety / distribution onto this code without the caller having to worry about it. Neat!

As you might imagine, there are quite a few problems that can be reduced to initialize a `WriteOnce<T>` this way, which vastly simplifies the process of making code thread safe and/or distributing it. If you see why that is, then you also already know why consensus is so foundational to distributed systems &mdash; because, drumroll please . . .

## Consensus Algorithms Implement Write-Once Variables

The word "consensus" means "agreement." Calling something consensus implies a sort of story: once upon a time, there was a group, where not all members of the group initially agreed. Then, they all came into agreement. We call that kind of agreement, "consensus."

In the context of a distributed system, the group is made up of threads running on different machines, rather than people, and the initial disagreement comes from different threads calling `tryInitialize` with different values. The final agreement comes from all threads receiving the same value from `finalValue`. If you check our write-once-based solutions from the previous heading, you'll see this in action. In both cases we start out with different threads in disagreement (different values are being passed to `tryInitialize`), and we end up with all threads in agreement (we forget what value we passed to `tryInitialize` and just trust the value returned by `finalValue` instead). Somewhere between `tryInitialize` and `finalValue`, the disagreement was resolved (arbitrarily, by picking the first `tryInitialize` call), and the final result was total agreement acrss all threads &mdash; consensus.

So, a write-once variable is a primitive backed by a consensus algorithm, which is to say a consensus algorithm is the algorithm that implements a write-once variable. They are two sides of the same coin.

That's a good high-level description, but we should dig deeper. What guarantees does a consensus algorithm need to provide? We should be able to glean everything we need from re-examining how the two example problems were built on top of `WriteOnce<T>`, and what implicit guarantees they were expecting `WriteOnce<T>` to provide.

## Properties of a Consenus Algorithm

Let's take a second look at the `WriteOnce` interface that a consensus algorithm must implement:

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable haa been initialized */
  public Future<T> finalValue();
}
```

What hidden assumptions exist inside of this interface? In other words, what rules govern the relationship between what gets passed to `tryInitialize` and what is returned by `finalValue`? What behaviors, if not maintained, would break our write-once-based example solutions?  We will consider any such rule a property of a consensus algorithm.





TODO we should be able to frame this as an examination of hidden assumptions in the interface

* Validity: that finalValue() has to be a value that someone tryInitialized
* Agreement: all futures always resolve to the same value
  * Has both a spatial requirement (for all threads) and a temporal requirement (for all calls)
  * Legalese: if finalValue returns a value, then no other call to finalValue anywhere else in the system has returned or will return any other value
* Termination: the future must resolve eventually

TODO the segue out to fault tolerance: `DistributedWriteOnce<T>` already satisfies all of the above. Observe that means we have already done the impossible in creating a valid consensus algorithm. Something is wrong. A requirement is missing. Old content to potentially adapt here:

> That's right, `DistributedWriteOnce<T>` is a full-blown consensus algorithm!
>
> Anyways, we've managed to write a consensus algorithm already &mdash; namely, `DistributedWriteOnce<T>`. We could end this blog post right here if we wanted to. (And if we did, we'd be quitting while we're ahead.) But clearly this cannot be the whole story. We managed to write a consensus algorithm already, without all that much thinking or code; aren't consensus algorithms supposed to be notoriously difficult for mere mortals to comprehend? I'm a mere mortal, and I comprehend `DistributedWriteOnce<T>` just fine . . .

## The Curveball: Fault Tolerance

TODO old content to adapt here:

> The new dimension of problems distributing adds on top of multithreading is dealing with **faults** in the system: hardware can lose power, software can crash, and networks can degrade or become disconnected. Technically these problems also affect single-node systems, but single-node code generally doesn't care about these faults because, if one occurs, then the node has failed and the code is no longer running. In other words, these faults do cause a mess, but nobody expects single-node code to be able to deal with that mess because the computer running the code is down. In a distributed system, a fault can affect a subset of the system's nodes while leaving code running on the remaining nodes to deal with the mess.
>
> Both our distributed solutions to the exactly-once and happened-or-not problems rely on a "coordinator" node to make progress, and neither can deal with a fault bringing the coordinator down: the bully algorithm will continue to pick that node as the coordinator, even though it's not online and can't do its work. In the exactly-once solution, the coordinator is the only node allowed to call `thingy()`, so if the coordinator crashes we never call `thingy()` and thus fail in our ultimate goal to call `thingy()` exactly once. In happened-or-not, the coordinator is the only node that stores the current cancellation state, so if it crashes, other nodes will either fail or hang in their `tryShip` / `tryCancel` calls. Neither of our distributed solutions so far is **fault tolerant**.
>
> The lack of fault tolerance is certainly a limitation, but it may not actually be a problem! If you're operating a small network of nodes, in a highly controlled environment such as a data center, and you have the option to regularly schedule "maintenance windows" for taking the whole system offline and doing upgrades, then non-fault-tolerant distributed algorithms may work for you. As we have already seen, non-fault-tolerant distributed algorithms are often pretty simple, much more so than any of the algorithms we're going to go on and build in the rest of this article. However, at a certain network size it becomes infeasible to ensure all nodes are always online, and in many Internet-facing services it is not feasible to take the system down for maintenance, ever. In these situations, the only option is to design software that can tolerate faults that affect a subset of the system. Consensus algorithms are usually employed by people trying to make fault-tolerant systems, so for the rest of this article, we will assume fault tolerance is a non-negotaible requirement.
>
> It turns out to be very difficult to come up with fault-tolerant solutions to the exactly-once and happened-or-not problems. However, there is a nifty way to factor out the fault tolerance part of the problems, such that we build one fault-tolerant primitive and use it to solve these completely different problems (and many others). That fault-tolerant primitive is called a **consensus algorithm**.

## Implementing Fault-Tolerant Consensus

TODO this is entirely a lead up to section 2. Point out that having a coordinator is a problem, we must become peer-to-peer. What real life algorithms do you know groups can use to come to consensus? Oh yeah, voting. Maybe we can write a voting algorithm?

# Part 2: Voting



---

TODO

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

Part 5 is about moving from the single-shot core Synod algorithm to a general state machine replication primitive (i.e. write-once variable to mutable variable semantics). 

* In broad strokes
  * Replicate a log where each entry is individually a write-once consensus variable
  * Replaying the log deterministically rebuilds the state of any data structure
  * Paxos includes further optimizations for this situation (multi-paxos)
* We should boost Raft here
  * The Paxos papers provide a broad strokes outline of how to do SMR but don't solve it completely
  * The exercises left ot the reader are actually pretty hard
  * Raft is a state machine algorithm, without the Paxos single-shot core, and is more explainable
* So I will not delve into multi-paxos, but here is my recommendation
  * If you want a state machine replication, probably starting with Raft is the best way to do it today
  * However, the synod algorithm is worth understanding because it's small and fits nicely into other systems
    * As a way of replicating writes to a database, for example
