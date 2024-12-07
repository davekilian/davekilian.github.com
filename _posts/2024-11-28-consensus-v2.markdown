---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

Distributed consensus algorithms are a critical piece of modern computing infrastructure, but few people really understand how they work. When the first consensus algorithm, *Paxos*, was introduced to the world in 1989, it was met with a sort of [indifferent confusion](https://www.microsoft.com/en-us/research/publication/part-time-parliament/), and it wasn't until 9 years later that it had gained enough grassroots popularity to be published in a major journal. As the world moved online and everyone started building their critical infrastructure as distributed systems, Paxos found its way into the foundation of almost every one of those newly built systems. But, at the same time, it also grew notorious for being beyond the comprehension of mere mortals &mdash; so much so that, when a second viable consensus algorithm, *Raft*, finally came along well over a decade later, the paper was called *In Search of an Understandable Consensus Algorithm*.

I like Raft, and I think if you're choosing between Raft and multi-Paxos for a project today, Raft is likely the better choice. Multi-Paxos is presented only as a rough sketch in the original Paxos paper, and although a hodgepodge of other papers try to fill in the blanks, they aren't completely consistent with one another. In the end, you'll find yourself partially reinventing the wheel. At least with Raft, there's a complete algorithm  in one place! Even so, I don't think Paxos should be skipped entirely. It's true the full multi-Paxos algorithm is complex and under-specified, but the core of Paxos (a little protocol called the *synod algorithm*) is small, well-specified, insightful, and can be embedded into other systems and algorithms in a way that the full Raft and multi-Paxos cannot. (TODO: Find an example. I thought Aurora does some kind of row-level Paxos using the synod algorithm? Am I misremembering?)

Leslie Lamport, who invented Paxos, [once described Paxos's core synod algorithm](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) as "among the simplest and most obvious of distributed algorithms," and claimed that it "follows unavoidably from the properties we want it to satisfy." He's right, although it might not seem that way the first time you read through it. The Paxos synod algorithm falls out of a lot of invisible context: you need to really understand the problem we are solving, why obvious solutions fail, and what must be done to work around those failures, before the synod algorithm really does start to look like a clear and minimal solution. So that's what we'll do: let's take a fresh look at the problem of consensus, try to solve it, get stuck, and then find a workaround for our problem; we should end up reinventing Paxos’s core synod algorithm. Along the way we'll cover the FLP result, which bridges the gap from obvious algorithms that don't work, to algorithms (like Paxos) that do.

# Part 1: Consensus

Just about every distributed system has a consensus algorithm running it in somewhere. Even if you’ve never used in directly, the fancy distributed replicated database you're using itself relies on a consensus algorithm, as do any cloud services you use. But what makes consensus so fundamental? Why are they ubiquitous in distributed systems?

One of the fun (or maybe "fun") parts of programming distributed systems is that quite a few common programming tasks that are easy to write in normal code turn out to be really, really hard in distributed code. Consensus algorithms help turn problems like that back into something more tractable. To see what I mean, let's look at a couple examples where a consensus algorithm might be helpful:

## Example: Picking a Random User

Let's say we're writing code for a message board website, and we want to add a 'user of the day' function where we spotlight one particular user at random each day. How do write code to pick a user once, and only once each day?

A basic answer might be something like this:

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

For us as programmers, what's different about distributed systems is how threads communicate: on a single machine, threads can just share variables, but once we have different threads on different machines we need to use the network. Our multithreaded solution does rely on sharing variables, so we'll have to switch to using the network instead.

Here's one way to do that: we can pick a node arbitrarily and assign it the job of managing the user of the day. Let's call that node the **coordinator**. The coordinator decides who is the user of the day, and remembers that decision all day. Any time another node needs to know who is the current user of the day, that other node asks the coordinator.

Sounds great, but how do the nodes figure out which one is the coordinator? Well, for most distributed systems running in data centers or the cloud, we know ahead of time the full set of nodes that are going to participate in the system. If we provision each node with a copy of that node list, we could have all nodes independently run the same deterministic algorithm on that list to pick the coordinator. As long as all nodes run the same deterministic algorithm with the same node list as input, they'll all end up picking the same coordinator &mdash; without ever sending a single network message!

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

On startup, if a node runs the bully algorithm and realizes it itself is the coordinator, it starts serving the remote method call interface, using our existing multithreaded implementation:

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

In server code, lots of things happen all at the same time; sometimes two users try to do two conflicting things at the same time. How do we make sure we accept one user's action and reject the other? For example, let's say we have an online ordering system where an order, once placed, can be cancelled up until it is shipped from the warehouse. What if someone in the warehouse tries to mark the order as shipped (no longer cancellable) at exactly the same time the customer tries to cancel the order?

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

How about turning this into a distributed solution? We can do it the same way we did before: expose the implementation above as an RPC call from some coordinator node, and have all other nodes call into the coordinator when they need to get or update the order state. The client side might look like this:

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
  
  /** Reads the value once the variable has been initialized */
  public Future<T> finalValue();
}
```

The way we implement this thing should look pretty familiar by now. First we write a single-threaded implementation:

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

The idea here is to actually choose *many* random users each day, but only install the first such user as today’s user of the day, by leveraging the write-once semantics of the `tryInitialize()` method. (To complete the solution, we would need a way to get a new `WriteOnce<User>` each day, but that's not important for this discussion so I'm going to handwave it.)

Do you see how the multithreaded / distributed guarantees of `WriteOnce<T>` are automatically conferred onto this implementation of `getUserOfTheDay()`, without us having to do anything special? If we pass in a thread-safe `WriteOnce`, that makes `getUserOfTheDay()` thread-safe automatically, and if we pass in a distributed `WriteOnce` then `getUserOfTheDay` is also automatically distributed. Pretty nifty, huh?

Now let's try the order cancellation problem. For that one, the core question was how to determine a winner if two users try to take conflicting actions at the same time. One possible solution is to have all attempts to ship or cancel an order race to be the first to call `tryInitialize` on a write-once variable; whichver is processed first is the one we accept, and all subsequent attempts are rejected implicitly when `tryInitialize` ignores the corresponding write:

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

As you might imagine, there are quite a few problems that can be reduced to initializing a `WriteOnce<T>` this way, which in turn vastly simplifies the process of making that code thread safe and/or distributing it. If you see that, then you also already know why consensus is so foundational to distributed systems &mdash; because, drumroll please . . .

## Consensus Algorithms Implement Write-Once Variables

A write-once variable is a primitive implemented by a consensus algorithm, which is to say, a consensus algorithm is the algorithm that implements a write-once variable. The write-once variable is an interface, and a consensus algorithm is the implementation of the interface. They are two sides of the same coin.

The naming might seem a little odd. The word "consensus" means "agreement." Calling something consensus implies a sort of story: once upon a time, there was a group, where not all members of the group initially agreed. Then, they all worked it out and ended up in unanimous agreement. In this case, we call that kind of agreement, "consensus."

In the context of a distributed system, the group is made up of threads running on different machines, rather than people, and the initial disagreement comes from different threads calling `tryInitialize` with different values. The final agreement comes from all threads receiving the same value from `finalValue`. If you check our write-once-based solutions from the previous heading, you'll see this in action. Both start with different threads in disagreement (different values are being passed to `tryInitialize`), and we end up with all threads in agreement (we forget what value we passed to `tryInitialize` and just trust the value returned by `finalValue`). Somewhere between `tryInitialize` and `finalValue`, the disagreement was resolved (arbitrarily, by picking the first `tryInitialize` call), and the final result was total agreement acrss all threads &mdash; so we call it “consensus.”

So, a consensus algorithm implements a write-once variable. That's a good high-level description, but we should dig deeper. What guarantees does a consensus algorithm need to provide? We should be able to glean everything we need from re-examining how our two example solutions were rebuilt on top of `WriteOnce<T>`, and carefully checking what guarantees our code was implicitly expecting `WriteOnce<T>` to provide.

## Properties of a Consenus Algorithm

Let's take a second look at the `WriteOnce` interface a consensus algorithm must implement:

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value once the variable has been initialized */
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

Let's write a consensus algorithm where nodes throw out proposals and vote on them, just like in the movie example above. However, this won't be messy like the real world, where people have preferences; each of our nodes will just vote for whatever option it heard about first, and never change its mind. Let's try it out:

## Basic Voting Algorithm

To get started, we'll implement `tryInitialize()` by broadcasting a message to every node in the system, offering the caller's value as a candidate for voting on:

```java
public void tryInitialize(T value) {
  for (int nodeId : App.nodeIds()) {
    remoteCall(nodeId, "onCandidate", value);
  }
}
```

When a node receives one of these messages for the first time, it votes for that value, and also broadcasts a message to its peers saying it voted for that candidate. After this point, the node is no longer allowed to change its vote, so if more candidates are received we silently ignore them, which is achieved by the `isEmpty()` check below:

```java
private void onCandidate(T value) {
  synchronized (lock) {
    if (myVote.isEmpty()) {
      myVote = Optional.of(value);
      for (int nodeId : App.nodeIds()) {
        remoteCall(nodeId, "onVote", value);
      }
    }
  }
}
```

Finally, each time a node receives a notification that a peer has voted, it increases its local tally of votes for candidate values. Once it sees some candidate has received a majority (more than half) of votes, it deems that value to be the winner, and sends that value to all pending and future `finalValue()` calls.

```java
CompletableFuture<T> outcome = new CompletableFuture<>();

// ...

private void onVote(T value) {
  synchronized (lock) {
    tally.put(value, 1 + tally.getOrDefault(value, 0));
    if (tally.get(value) > App.nodeIds().size() / 2) {
      outcome.complete(value);
    }
  }
}
```

Finally, we implement `finalValue()` just by passing along the outcome future:

```java
public Future<T> finalValue() {
  return outcome;
}
```

Putting it all together, we get a `WriteOnce<T>` implementation that looks like this:

```java
class MajorityRulesVoting<T> implements WriteOnce<T> {
  private Object lock = new Object();
  private Optional<T> myVote = Optional.empty();
  private Map<T, Integer> tally = new HashMap<>();
  private CompletableFuture<T> outcome = new CompletableFuture<>();
    
  public MajorityRulesVoting() {
    registerRemoteCall("onCandidate", this::onCandidate);   
    registerRemoteCall("onVote", this::onVote);
  }
  
  public void tryInitialize(T value) {
    for (int nodeId : App.nodeIds()) {
      remoteCall(nodeId, "onCandidate", value);
    }
  }
  
  private void onCandidate(T value) {
    synchronized (lock) {
      if (myVote.isEmpty()) {
        myVote = Optional.of(value);
        for (int nodeId : App.nodeIds()) {
          remoteCall(nodeId, "onVote", value);
        }
      }
    }
  }
  
  private void onVote(T value) {
    synchronized (lock) {
      tally.put(value, 1 + tally.getOrDefault(value, 0));
      if (!outcome.isDone()) {
        if (tally.get(value) > App.nodeIds().size() / 2) {
          outcome.complete(value);
        }
      }
    }
  }

  public Future<T> finalValue() {
    return outcome;
  }
}
```

Note in the implemenation above, the node running the code is itself included in the `App.nodeIds()` list, which means it always sends `onCandidate` and `onVote` messages to itself via the network in addition to informing its peers. I did it this way mostly to save on space so the code would fit nicely in a page of this blog post.

One important caveat with this implementation: to be fault tolerant, we actually need to keep broadcasting `onCandidate` and `onVote` messages forever instead of just sending each message once, so that nodes which were offline for a while and come back can learn of candidates and votes and thereby catch up. For the sake of keeping the code snippets clean, let's just pretend our `remoteCall()` interface internally rebroadcasts messages until they are delivered.

With that, it seems we have a complete `WriteOnce<T>`. So does this meet our needs? Let's check:

**Agreement**: ✅

If a candidate value receives more than half of the votes, then less than half of the votes remain for other candidates, which means only one candidate can reach a majority can become the `finalValue()` of the system. Every node ends up tallying the same set of votes. so they all independently reach the same determination of which candidate wins.

**Validity**: ✅

The final value is the value which received a majorify of the system's votes. A value can only receive any votes if it was a candidate, which in turn can only happen if someone passed that value to `tryInitialize`. Therefore the final value is always a value that was passed to `tryInitialize`.

**Termination**: ❌

We only terminate when a candidate value receives a majority of votes. Is that guaranteed to eventually happen? I don't think so.

## Split Votes

Majority-rules voting (as a system, not just our algorithm) is vulnerable to a situation called a **split vote**, where no candidate receives a majority of the votes, even once all votes have been cast and counted. Our algorithm is prone to the same problem: it is possible for every node to cast its vote, and still not have a winner.

For example, say we have a network with 25 nodes, and we are voting on one of three different colors: red, green or blue. If all three proposals (`tryInitialize` calls) for all three colors are made at around the same time, we could end up with the following tally:

<div class="overflows" markdown="block">

| Candidate Value | Number of Votes | Winner? |
| --------------- | --------------- | ------- |
| Red             | 9               | No      |
| Blue            | 5               | No      |
| Green           | 11              | No      |

</div>

Here no candidate is the winner because it takes at least 13 votes to reach a majority, and no candidate received 13 votes. Since we do not have a winner, the `finalValue()` future has not resolved. However, since all votes have been cast, the tally will never change either. So we have not finished and also cannot progress. We're deadlocked!

We'll have to find some kind of workaround. That's okay, this was our very first stab, it was likely we would run into some kind of stumbling block. But how we do deal with split votes?

One thing we could do is add the notion of a "do-over" to our algorithm: if we get to the end of voting and determine it's a split vote, just start over and hold a new vote. We might have more luck and reach a majority the next time around. This is not a bad idea, but I want to avoid it if possible for a few reasons. First, having restarts in a consensus algorithm requires great care: if the algorithm does manage to reach consensus, then restarts and reaches consensus again with a different answer, we end up violating the Agreement property. Second, we are solving the problem of not guaranteeing termination, and retrying doesn't guarantee termination anyways. We can't even put a reasonable bound on how many times we will have to retry before we can terminate.

A different approach that avoids both of these problems is to have a **tiebreaker**: just add a deterministic rule that chooses a winning value arbitrarily, and run it once all votes are in. Every node has the same tally of votes, so if every node runs the same deterministic rule over that tally to pick a winner, every node will pick the same winner.

## Tiebreaking

Let's take another look at the example tally from the previous section:

<div class="overflows" markdown="block">

| Candidate Value | Number of Votes | Winner? |
| --------------- | --------------- | ------- |
| Red             | 9               | No      |
| Blue            | 5               | No      |
| Green           | 11              | No      |

</div>

Looking at the results, what seems like a good tiebreaking rule?

The first one that jumps out to me is "pick the one that got the most votes," which is Green in the example above. In other words, instead of *majority* voting, implement *plurality* voting, where the candidate that got the most votes wins even if it didn't manage to reach a majority.

Plurality voting would work in the example above, but it doesn't work in this case:

<div class="overflows" markdown="block">

| Candidate Value | Number of Votes | Winner? |
| --------------- | --------------- | ------- |
| Red             | 10              | No      |
| Blue            | 10              | No      |
| Green           | 5               | No      |

</div>

Here we have two winning candidates but plurality voting rules; to narrow it down further, we would need another tiebreaking rule (a tiebreaker for the tiebreaker). But, whatever rule we use as the tiebreaker-of-tiebreakers would probably work as a tiebreaker in its own right. So let's not use plurality voting.

I think we've actually already discussed a rule that we could use as an alternate tiebreaker here: it's the bully algorithm! Why don't we take all the candidates that got any votes, sort them in some arbitrary order, and then pick the first or last one in the sort order?

```java
class Tiebreaker<T extends Comparable> {
  T tiebreak(Map<T, Integer> tally) {
    return Collections.max(tally.keySet());
  }
}
```

I think this'll work. Once integrating this into our original voting algorithm, we get an implementation something like this:

```java
class TiebreakVoting<T extends Comparable> implements WriteOnce<T> {
  private Object lock = new Object();
  private Optional<T> myVote = Optional.empty();
  private Map<T, Integer> tally = new HashMap<>();
  private int votesTallied = 0;
  private CompletableFuture<T> outcome = new CompletableFuture<>();
    
  public MajorityRulesVoting() {
    registerRemoteCall("onCandidate", this::onCandidate);   
    registerRemoteCall("onVote", this::onVote);
  }
  
  public void tryInitialize(T value) {
    for (int nodeId : App.nodeIds()) {
      remoteCall(nodeId, "onCandidate", value);
    }
  }
  
  private void onCandidate(T value) {
    synchronized (lock) {
      if (myVote.isEmpty()) {
        myVote = Optional.of(value);
        for (int nodeId : App.nodeIds()) {
          remoteCall(nodeId, "onVote", value);
        }
      }
    }
  }
  
  private void onVote(T value) {
    synchronized (lock) {
      tally.put(value, 1 + tally.getOrDefault(value, 0));
      votesTallied++;
      
      if (!outcome.isDone()) {
        if (tally.get(value) > App.nodeIds().size() / 2) {
          // This value has reached a majority
          outcome.complete(value);
        } else if (votesTallied == App.nodeIds().size()) {
          // All votes in with no majority, use bully algorithm
          outcome.complete(Collections.min(tally.keySet()));
        }
      }
    }
  }

  public Future<T> finalValue() {
    return outcome;
  }
}
```

How are we looking now?

**Agreement**: ✅

Case: some candidate value reaches a majority. In this case, each node learns of the majority winner before ever trying to run the tiebreak rule, so our analysis from before still holds. 

Case: no candidate reaches a majority. In this case, every node has the same tally and runs the same deterministic rule to pick a winner from among the candidates, so all nodes pick the same winner. Since we wait until all votes are in before running the tiebreaking rule, we cannot change our minds later.

**Validity**: ✅

Whether voting ends because a value reached majority or by the tiebreaking rule, we only ever select a candidate value which received at least one vote. To receive a vote, a candidate must be broadcast to `onCandidate`, which only happens if it was ever passed to `tryInitialize`. Thus the accepted value is always one that was proposed.

**Termination**: ✅

Case: some candidate value reaches a majority. Then the algorithm terminates once the majority is reached.

Case: no candidate reaches a majority. Then the algorithm terminates once all votes have been cast.

**Fault Tolerance**: ❌

Oops, this isn't fault tolerant, is it? We don't run the tiebreaker until all votes are in ... but if a fault takes down any node before that node casts its vote, then we'll never have all votes in, and then we won't terminate, will we?

## Fault-Tolerant Tiebreaking 

Okay, so we've realized we need a tiebreaker to force the algorithm to eventually terminate, and we now realize that tiebreaker needs to execute before all votes are in if we want it to be fault tolerant. So how do we do that?

... I'm really not sure.

Running the tiebreaker before all votes are in is dangerous: if more votes arrive after we run the tiebreaker, they can change the algorithm's answer and get everything all confused. For example, we could run the bully algorithm at a time where the tally only has votes for `Red` and `Blue`, and decide our answer is `Blue`; but then if other votes come in and even one of those votes picks `Green`, the next node to run its own tiebreaker may run the bully algorithm over `Red`, `Blue` and `Green` and pick `Green`. Then we have disagreement, the algorithm is broken.

Maybe we could work around this by having a different, even more clever tiebreaker than the one we already have. But I doubt it. Even if we had a tiebreaker that somehow couldn't change its mind, we'd still have to deal with the possibility that some candidate value reaches a majority after we run the tiebreaker. If that happens, any node that ran this clever tiebreaker still might end up disagreeing with nodes that saw the majority value first. So the algorithm is still broken.

What we really need to do is adjust the trigger condition for running the tiebreaker in the first place. We want the tiebreaker to run after all healthy nodes' votes are in, but without waiting for votes from faulted nodes whose votes will never arrive. But in practice, how do we know when that is?

In literature, a piece of code that can tell the difference between "the remote node is still running, but it's not done yet" and "the remote node has failed, and will never do its work" is called a **failure detector**. Quite a bit of theory has been built around what we can do if we have a reliable failure detector, but the fact that remains, failure detectors don't exist in the real world. (At least, not 100% accurate failure detectors, which is what we need here.) Sure, we can detect that network requests we have sent to other nodes have timed out, but we can't tell the difference between "the reply hasn't arrived yet" vs "the reply is never coming." No matter how long we've waited, there's always a possibility the reply could arrive if we just waited a little longer.

Without a failure detector, there's no way fault-tolerant way to run a tiebreaker safely; finding the right time to run the tiebreaker would by definition be finding a perfect fault detector. So running the tiebreaker before all votes are in simply isn't an option. On the other hand, waiting until all votes are in isn't an option either &mdash; we would sacrifice fault tolerance. So tiebreaking rules are completely out.

## Alternatives to Tiebreakers?

Our basic majority voting algorithm doesn't always terminate if we run it without a tiebreaker. But we also can't run tiebreakers: either we end up with a broken algorithm, or we get an algorithm that sometimes deadlocks due to just one node crash. We need a lateral move. Can we fix split votes without a tiebreaker?

Well, early on we passed on the idea of restarting the algorithm if we get a split vote. Would that work? Well, once again, we have to decide whether we wait until all votes are in before triggering a revote; if we wait for all votes before triggering a revote then we can't sustain the loss of even one node, but if we don't wait for all votes, the old voting round could produce a majority after we called for a revote, resulting in inconsistency. In other words, revoting has the same problem as the other tiebreaking rules we tried before; in retrospect, revoting is itself just another tiebreaking rule.









TODO

* What if we tried to use timeouts to rule out bad nodes? That's more clever but it still doesn't work: the node that did the last vote might know it has reached consensus and resolved `finalValue()` futures even though it can't communicate out to anybody else, which is another but more subtle agreement violation.
* Both of these are good setup because the final Paxos solution is to make it safe to restart at any time. The point of synod is to guarantee, at any time, if someone successfully restarts the vote after consensus has been reached, always they will reach consensus again on the same value.
* This is also potentially a good segue out: observe everything else we do looks an awful lot like a tiebreaker. Proving this would be huge: it would show that we're actually overconstrained.

# Intermission

We set out with the goal of learning the problem space of consensus, so that we could see Paxos as the simplest possible solution to an unintuitive problem, rather than an unintuitive solution to a seemingly simple problem. Since then, we've gained an appreciation for why consensus is uniquitious in distributed systems, we nailed down the exact properties a consensus algorithm should have, and we've written quite a few consensus algorithms ourselves, between the `WriteOnce` implementation exercises in Part 1 and the voting algorithms in Part 2. Unfortunately, we have yet to invent a fault-tolerant distributed consensus algorithm, and as of yet we don't have any ideas to try next.

This is the low point in our journey. We understand the problem and we appreciate its difficulty, but we don't yet have a way forward. Things will only get better from here. We're about to turn our attention to the FLP result, which explains what we have been doing wrong so far, and in doing so we'll learn what we need to change about our approach in order to arrive at the Paxos algorithm. Pour yourself your stimulant drink of choice and let's keep going!

# Part 3: The FLP Result

TODO:: a good cold open might be a repeat of Lemma (2? 3?) the inscrutable one. "Refuses to elaborate, leave"

TODO: open as the proof generalizes what you already saw with the tie breaker / pick your poison



---

Part 3 is FLP

Basic proof structure from my old notes:

1. We terminate, so there is some step that always decides
2. It could decide either way, depends on the network delivery order on that node
3. If we lose that node, we need additional logic to decide without that node
4. Requirements for the additional logic
    * If the node is up and decides 0, the additional logic decides 0
    * If the node is up and decides 1, the additional logic decides 1
    * If the node is down, the additional logic does not wait on the node
    * We cannot distinguish between the node up and node down cases
5. Clearly this is impossible; we cannot be consistent with what the node decides without waiting to see what it decides. So we cannot write that additional logic. It does not exist.
6. But that means we do not terminate in the event of one fault. QED

Hookups to the actual proof, probably don’t need to harp on this but good to leave breadcrumbs 

1. This is event e
2. This is e’ and that p=p’
3. This is lemma 3’s finite deciding run
4. This is figure (3?), the big graph, plus the commutativity argument 
5. Conclusion of contradiction in lemma 3
6. The main proof of non-termination

Draw parallel between the lemma 3 finite deciding run and the tiebreaker before all votes are in. All other algorithms we have written, including distributed write once, basic voting, and tiebreaker at the end are all examples of non-terminating consensus algorithms that do have agreement and validity.

Finally, getting around this. We are over constrained, we must sacrifice agreement validity or termination. Termination: non-guaranteed termination is very different from guaranteed non-termination. Maybe we can rely on the probability of termination continually increasing over time. Like if you have a 10% chance of failing a round, 1 in 10 executions proceed to the second round, 1 in 100 proceed to the third round, and one in 10 billion proceed to round 10. 

How do we do that? We need a voting algorithm with an infinite number of rounds, and also avoids ever having a deciding step. We’re starting to encroach onto liveness here

Part 4 is Paxos

* Key ideas
  * Voting progresses in rounds, potentially infinitely many rounds
  * In each round, the vote is one-sided: either we choose this specific candidate, or we remain undecided
* Show an implementation just called `Paxos<T> implements WriteOnce<T>` which implements the basic strategy with symmetric peers and not much code
* Add the bells and whistles to "grow" it into the synod algorithm you would recognize from Paxos Made Simple or Wikipedia

Part 5 is about state machine replication

* The cherry on top: making the `WriteOnce<T>` variable into a mutable variable with the same guarantees
  * This is what people actually usually mean when they say you need a consensus algorithm
  * ... I may need to adjust earlier sections to be really clear I'm using "consensus as write-once" sort of incorrectly
    * Well, Paxos people might agree with me, maybe it's controversial lol
* What does that mean semantically? Introduce state machine replication
  * If you have any data structure that can be built deterministically
  * And it has a deterministic starting point
  * And you have a list of operations you did on it
  * Then you can rebuild it by replaying the log
  * Database people call this a write-ahead log, file system people call it journaling
* Two ways of doing it
* The multi-Paxos way: each log entry is an instance of synod. Some protocol optimizations to reduce repeated work. Electing one leader is a big one
* The Raft way: build state machine replication as a first class primitive. Leader election uses our basic voting algorithm, but with randomized timeouts for retry; it's not an agreement violation to bounce the leader around. Within a leader epoch, we do log replication, and between epochs we do log recovery. There is no synod-like thing in here
