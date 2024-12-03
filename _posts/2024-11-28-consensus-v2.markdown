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
int bullyAlgorithm(Collection<Integer> nodeIds) {
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
  
  private int bullyAlgorithm(Collection<Integer> nodeIds) {
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

## Using Write-Once Variables

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

Let's take a second look at the `WriteOnce` interface a consensus algorithm must implement:

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable haa been initialized */
  public Future<T> finalValue();
}
```

What hidden assumptions exist behind this interface? In other words, what rules govern the relationship between what gets passed to `tryInitialize` and what is returned by `finalValue`? What expected behaviors, if violated, would the example solutions we built on top of this interface?  Any such rule is a property of a consensus algorithm.

Here's one I can see: since this is a write-once variable, its value must never change. So all futures returned by `finalValue()` must always resolve to the same value. Different threads must not get different values from `finalValue()`, nor can the same thread see different values over time. Think of the pandemonium that would ensue if we failed to uphold this rule, and in doing so we let the customer cancel an already-shipped order after the box was already on the truck!

In papers and textbook, this rule is sometimes called **agreement**:

> **Agreement**: If the consensus algorithm returns a value, then no other value has ever been or will ever be chosen. That is: if any call to `finalValue()` returns a value, no other call to `finalValue()` has ever or will ever return any other value.

Here's another rule I can see: the value returned by `finalValue` has to be the value somebody previously passed to `tryInitialize`. It can't be some other nonsense value, random value, or something picked arbitrarily. For example, an execution like this should not be allowed:

* `WriteOnce<Integer>` created
* Thread 1 calls `tryInitialize()` passing in a value of 1
* Thread 2 calls `tryInitialize()` passing in a value of 2
* Thread 3 calls `finalValue()`, which returns 4 [*](https://xkcd.com/221/)

That might seem obvious, but it's still worth writing down. In *the literature*, this rule is sometimes called **validity**:

> **Validity**: The value chosen by a consensus algorithm must be a value that was previously proposed. That is: the value returned by `finalValue()` must be a value previously passed to `tryInitialize()`. 

One final rule: once `tryInitialize()` has been called, the algorithm gets a reasonable amount of time to do its processing before `finalValue()` futures finally resolve to the chosen value. We can't wait on the future forever. This rule is called **Termination**:

> **Termination**: The algorithm eventually chooses some value; after `tryInitialize()` has been called, the futures returned by `finalValue()` eventually resolve

Agreement, Validity, Termination; seems like a pretty good starting set. However, I think we must still be missing a rule.

By the three rules above, `DistributedWriteOnce<T>` is a valid implementation of a distributed consensus algorithm. But consensus algorithms are supposed to be incomprehensible to mere mortals; I am a mere mortal, and I comprehend `DistributedWriteOnce<T>` just fine. There must be another thing that consensus algorithms do, that `DistributedWriteOnce<T>` does not.

## The Curveball: Fault Tolerance

Earlier, we said the main complexity that distributing an algorithm adds on top of multithreading is that threads in a distributed system must communicate through a network, since they can't share variables. When we said that, I glossed over a really important problem that introduces: it is impossible to communication via variable-sharing to fail, but you bet communiating over a network can.

Distribution introduces the problems of **faults** in the system. Hardware can lose power, software can crash, and networks can degrade or disconnect. Technically these problems also affect single-node software, but nobody ever expects single-node code to be able to deal with a fault: for example, if the machine running some code crashes, the code is no longer running, and thus can't do anything about it. Since single-node code is no longer running and can't do anything if the hardware faults, it can largely pretend faults don't exist. In a distributed system, a fault can affect some nodes while leaving others online to deal with the consequences.

Distributed system code that keeps working even if some nodes fault is called **fault tolerant**. The distributed consensus algorithm we previously implemented, `DistributedWriteOnce<T>` is not fault tolerant: it relies on a single coordinator node and cannot recover if a single fault happens to take down that coordinator node. This is because, in normal operation, all non-coordinator nodes contact the coordinator any time they need to get or set the variable:

```java
public void tryInitialize(T value) {
  return remoteCall(coordinatorId, "tryInitialize");
}

public Future<T> finalValue() {
  return remoteCall(coordinatorId, "finalValue");
}
```

There is no provision here ever to not use the coordinator node, so if the coordinator crashes, loses power or otherwise goes away, this code will continue to send messages to the remote coordinator and wait for a response. But the coordinator is gone, so no response ever comes, and so the algorithm fails to make progress. Since it took just one fault &mdash; one downed node &mdash; to break the `DistributedWriteOnce` consensus algorithm, it is not fault tolerant.

Now, the lack of fault tolerance is certainly a limitation, but depending on your use case that may not be a problem. Plenty of systems run on a small number of nodes, and if you have a small network with just a few machines, hardware and software is reliable enough that you won't see faults in the system very often. Rare hardware and software faults only become a consistent nuisance if you have a large enough system to start hitting rare problems frequently. Also, many systems do not need to be highly available: you can take them offline from time to time to do regular maintenance like upgrading software, replacing old hardware, etc.

If you're dealing with a small system that admits maintenance windows, you can probably get away with something like `DistributedWriteOnce<T>`, which is pretty nice: `DistributedWriteOnce<T>` is much simpler than anything else we're about to discuss, so really you'd be quitting while you're ahead. However, if you have a really big system, or you really can't accept any downtime at all (e.g. maybe you're running the city's E911 or something), then you require fault tolerance, and `DistributedWriteOnce<T>` is too simplistic to meet your needs.

Most of the time, when people talk about consensus algorithms, they mean fault tolerant consensus algorithms. So we are going to worry about making our consensus algorithms fault-tolerant too. Thus, on top of our Agreement, Validity and Termination rules, we'll add a fourth:

> **Fault Tolerance**: The presence of faults in hardware, software or the network do not cause the algorithm to violate the other properties (Agreement, Validity or Termination)

There is some wiggle room in choosing how many faults the algorithm needs to be able to survive. Obviously there is nothing the software can do if every machine in the network loses power simultaneously; typically you either give a constant number of node faults your system can tolerate (typically between 1 and 3) or you say some percentage of machines are allowed to fail ("less than half" is pretty common).

## Implementing Fault-Tolerant Consensus

Our previous strategy of implementing distributed consensus was not fault tolerant because it relied  a single coordinator node to be solely responsible for storing and managing the varible. If we want something fault tolerant, we will have to do away with the coordinator and come up with an entirely peer-to-peer algorithm. Every node needs to have its own copy of the variable, and we need to come up with some protocol for them all to come into agreement as to what value they should store.

Do you know any real-life leaderless, peer-to-peer algorithms for groups to come to consensus?

For example, think of a group of friends that want to go to the movies, and need to deicide what movie to see. That's an agreement problem. What might happen next? Maybe someone throws out an idea, someone throws out another, some people agree, others disagree, eventually a group opinion starts to form. The tide starts to turn when someone says

<center>"I vote we (blah blah blah . . .)"</center>

Interesting &mdash; majority-rules voting is an algorithm that does not require a leader, and does lead a disagreeing group into agreement. Maybe we could code voting into a consensus algorithm? Let's try it.

# Part 2: Voting





TODO

```java
class MajorityRulesVoting<T> implements WriteOnce<T> {
  private Object lock = new Object();
  private Optional<T> myVote = Optional.empty();
  private Map<T, Integer> voteCounts = new HashMap<>();
  private CompletableFuture<T> outcome = new CompletableFuture<>();
    
  public MajorityRulesVoting() {
    registerRemoteCall("voteFor", this::voteFor);   
    registerRemoteCall("onResult", this::onResult);
  }
  
  public void tryInitialize(T value) {
    for (int nodeId : App.nodeIds()) {
      remoteCall(nodeId, "voteFor", value);
    }
  }
  
  private void voteFor(T value) {
    synchronized (lock) {
      if (myVote.isEmpty()) {
        myVote = Optional.of(value);
        for (int nodeId : App.nodeIds()) {
          remoteCall(nodeId, "onResult", value);
        }
      }
    }
  }
  
  private void onResult(T value) {
    synchronized (lock) {
      voteCounts.set(value, 1 + voteCounts.getOrDefault(value, 0));
      if (voteCounts.get(value) > App.nodeIds().size() / 2) {
        outcome.complete(value);
      }
    }
  }

  public Future<T> finalValue() {
    return outcome;
  }
}
```





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
