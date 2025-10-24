---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

I would venture to say programmers generally know two things about consensus algorithms: that they're important for cloud computing at a fundamental level, and that basically nobody knows how they work &mdash; in fact, most people can't say what a consensus algorithm does, or what fundamental role it would play in a larger system. You can work on cloud software all your life and never interact with a consensus algorithm even once, yet everything you build in the cloud ends up relying on consensus algorithms, because there are consensus algorithms running somewhere behind all the APIs and abstractions cloud software is built on. This article itself was brought to you in part by a consensus algorithm.

When the first consensus algorithm, *Paxos*, was introduced to the world in 1989, it was met with a sort of [indifferent confusion](https://www.microsoft.com/en-us/research/publication/part-time-parliament/), and it wasn't until 9 years later that it had gained enough grassroots popularity to be taken seriously. As the world moved online and distributed systems became the way we build software, Paxos found its way into the foundation those new systems. But at the same time, it also grew notorious for being beyond the comprehension of mere mortals. In fact, when a second viable consensus algorithm, *Raft*, finally came along well over a decade later, the paper was called *In Search of an Understandable Consensus Algorithm* &mdash; the accomplishment was not in creating a new consensus algorithm, but rather in being able to explain how one works!

These days, Raft has mostly supplanted Paxos. You can roughly estimate the age of a distributed systems project by the consensus algorithm it uses: if it uses Raft, development probably started in the last ten years or so; if it uses Paxos, it’s older. Raft is a “batteries included” paper: if you implement exactly what you find in the paper, you will end up with a full consensus system which works the way programmers usually expect a consensus system to work. For Paxos, the original papers present the full system only as a sketch, which means everybody implementing Paxos has to fill in those blanks themselves. Needless to say, every implementation of Paxos is a little bit different. If you’re starting a project today and you need to implement a distributed consensus algorithm, Raft is probably the better choice. 

Even so, I don't think Paxos should be skipped entirely. The kernel at the core of Paxos is a little protocol called the *synod algorithm*, which is small, well-specified, insightful, and can be embedded into other systems and algorithms in a way that Raft and the full ‘multi-Paxos’ algorithm cannot. Whether or not you ever design a full Paxos, I think the synod algorithm is worth learning.

Leslie Lamport, who invented Paxos, [once described Paxos's synod algorithm](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) as "among the simplest and most obvious of distributed algorithms," and claimed that it "follows unavoidably from the properties we want it to satisfy." That is certainly true, but you should know the algorithm falls out of a lot of invisible context: for the synod algorithm to seem obvious, you have to really understand the problem we are solving, and how the intuitive solutions are all broken. So let's do exactly that: let's figure out what exactly a consensus algorithm should do, try to solve the intuitive way, find out the intuitive solutions are all broken, and then get stuck; in getting oursleves un-stuck, we'll have to figure out what was wrong with the intuitive algorithms, and in correcting our approach, we should be lead into the synod algorithm "unavoidably."

# Part 1: Consensus

Just about every distributed system has a consensus algorithm running it in somewhere. Even if you’ve never interacted with a consensus algorithm directly (most programmers haven’t), you have still relied on one if you’ve ever built anything in the cloud. Just about every cloud service relies on a consensus algorithm in one way or another.

But what makes consensus so fundamental? Why are they ubiquitous in distributed systems?

One of the fun (or maybe "fun") parts of programming distributed systems is that quite a few common programming tasks that are normally easy turn out to be really, really hard in distributed code. Arguably the most important of those is making something happen just once. Normally, making something happen once is the default: just write code to do it, and don’t put it in a loop. But in distributed systems, making something happen “just once” turns out to be *fiendishly* difficult. Consensus algorithms provide a generic solution for turning “just once” problems back into something more tractable.

---

TODO - one thing I don’t like here is I made it sound like the problem solved is “exactly once,” but really there are an interconnected set of problems which are mutually reducible to one another without much effort:

* Exactly once
* Conflict resolution
* Ordering events

The relationship between these problems is interesting but not something we can talk about here / we may not have space to at all. However, we should make it clear there are several problem classes that are all served by consensus (readers should start being able to identify what they are)

Luckily we already have two problems which fall easily under the two classes -- exactly once for user of the day, conflict resolution for order statuses, we just need to rebrand them as different problems with surprisingly similar solutions.

---

To see what I mean, let's look at a couple examples where a consensus algorithm might be helpful, even for small tasks that ought not to be too difficult.

## Example: Picking a Random User

Let's say we're writing code for a message board website, and we want to add a 'user of the day' feature where we spotlight one user chosen at random each day. How do write code to pick a user once, and only once each day?

In real life, we’d probably just add a field to some kind of database, but let’s pretend we don’t have a database because modern distributed computing infrastructure hasn’t been invented yet. For this exercise, all we have are computers, networks, and our code. The problem should still fundamentally be solvable, even if we have to do the heavy lifting ourselves. So without fancy infrastructure, how do we pick a user once each day?

If we have everything running on one machine, a simple answer might be something like this:

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

For us as programmers, what's different about distributed systems is how threads communicate: on a single machine, threads can just share variables, but once we have different threads on different machines we need to use the network. We do indeed rely on sharing variables in our multithreaded solution, so to distribute it we'll have to switch to using the network instead.

Here's one way to do that: we can pick a node arbitrarily and assign it the job of managing the user of the day. Let's call that node the **coordinator**. The coordinator decides who is the user of the day, and remembers that decision all day. Any time another node needs to know who is the current user of the day, that other node asks the coordinator.

---

TODO: can we safely remove the bully algorithm discussion from this part of the article?

I’m not sure it really serves the overall narrative, and until we start talking about failures and fault tolerance, there is no obvious reason not to simply provision each node with a hardcoded coordinator address.

---

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

On startup, if a node runs the bully algorithm and realizes it itself is the coordinator, it starts serving the remote method call interface, backed by our previous multithreaded implementation:

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

Once again this is simple enough to solve in regular, run-of-the-mill single-threaded code. We just need a variable with three states: order pending (not yet shipped, can still be cancelled), cancelled (no longer can be shipped), and shipped (no longer can be cancelled):

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

It's interesting that we have two different problems, both with completely different single-threaded solutions, which could nonetheless be multithreaded and then distributed using the exact same approaches. In fact, there is an opportunity for refactoring here: we can isolate the code that does the multithreaded / distributed parts of these solution and thereby factor it out of the main solution code, by introducing a new abstraction which I'll call a *write-once variable*.

## Write-Once Variables

Say we have a primitive I'll call a **write-once variable**. The interface looks something like this:

```java
interface WriteOnce<T> {
  /** Try to initialize to the given value, no-op if already initialized */
  public void tryInitialize(T value);
  
  /** Reads the value, or null if not yet initialized */
  public T finalValue();
}
```

The way we implement this thing should look pretty familiar by now. First we write a single-threaded implementation:

```java
class BasicWriteOnce<T> implements WriteOnce<T> {
  private T value = null;
  private boolean initialized = false;
  
  public void tryInitialize(T value) {
    if (!this.initialized) {
      this.value = value;
      this.initialized = true;
    }
  }
  
  public T finalValue() {
    return this.value;
  }
}
```

To make it thread-safe for multithreading, we just add a lock:

```java
class MultithreadedWriteOnce<T> implements WriteOnce<T> {
  private Object lock = new Object();
  private T value = null;
  private boolean initialized = false;
  
  public void tryInitialize(T value) {
    synchronized (this.lock) {
      if (!this.initialized) {
        this.value = value;
        this.initialized = true;
      }
    }
  }
  
  public Future<T> finalValue() {
    return this.value;
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
      return impl.finalValue();
    });
  }
}
```

Once again we were able to multithread the initial solution with a lock and distribute the multithreaded solution by selecting a coordinator node. Now let's see how rewriting our other two solutions on top of the `WriteOnce<T>` abstractions lets us factor away all the details involving multithreading and distribution.

## Using Write-Once Variables

Let's start with the random user-of-the-day problem. The core question behind that one was how to make something happen once; in this case, we wanted to make sure one, and only one random user is picked each day.

Here's a way to do that with a write-once variable:

```java
User getUserOfTheDay() {
  User randomUser = User.randomUser();
  
  WriteOnce<User> userOfTheDay = // ...
  userOfTheDay.tryInitialize(randomUser);
  return userOfTheDay.finalValue();
}
```

The idea here is to actually choose *many* random users each day, but only pick the first such user to become the user of the day, by virtue of being the only randomly chosen user to initialize the write-once variable. (To complete the solution, we would need a way to get a new `WriteOnce<User>` each day, but that's not important for this discussion so I'm going to handwave it.)

Do you see how the multithreaded / distributed guarantees of `WriteOnce<T>` are automatically conferred onto this implementation of `getUserOfTheDay()`, without us having to do anything special? If we pass in a thread-safe `WriteOnce`, `getUserOfTheDay()` becomes thread-safe automatically, and if we pass in a distributed `WriteOnce` then `getUserOfTheDay` is also automatically distributed. Pretty nifty, huh?

Now let's try the order cancellation problem. For that one, the core question was how to resolve a conflict between two users’ concurrent actions; in this case the conflict was between the warehouse shipping the order and the user cancelling it. One possible solution is to have all attempts to ship or cancel an order race to be the first to call `tryInitialize` on a write-once variable; whichever is processed first is the one we accept, and all subsequent attempts are rejected implicitly when `tryInitialize` ignores the corresponding write:

```java
WriteOnce<OrderState> orderResult = // ...

void tryCancel() {
orderResult.tryInitialize(OrderState.CANCELLED);
}

void tryShip() {
  orderResult.tryInitialize(OrderState.SHIPPED);
}

OrderState getOrderState() {
  OrderResult result = orderResult.finalValue();
  if (result == null) {
    return OrderState.PENDING;
  }
  
  return result;
}
```

Once again, the internal guarantees of thread safety / distribution provided by the `WriteOnce<OrderState>` confer thread safety / distribution onto this code without the caller having to worry about it. Neat!

As you might imagine, there are quite a few problems that can be reduced to initializing a `WriteOnce<T>` this way, which in turn vastly simplifies the process of making that code thread safe and/or distributing it. If you see that, then you also already know why consensus is so foundational to distributed systems &mdash; because, drumroll please . . .

## Consensus Algorithms Implement Write-Once Variables

A write-once variable is a primitive implemented by a consensus algorithm, which is to say, a consensus algorithm is the algorithm that implements a write-once variable. The write-once variable is an interface, and a consensus algorithm is the implementation of the interface. They are two sides of the same coin.

The naming might seem a little odd. The word "consensus" means "agreement." How does initializing a write-once variable involve agreement?

Maybe we should take a closer look at the definition of consensus first. Calling something consensus implies a sort of story: once upon a time, there was a group, where not all members of the group initially agreed. Then, they all worked it out and ended up in unanimous agreement. In this case, we call that kind of agreement, "consensus."

Implementing a distributed write-once variable internally involves that story of agreement. Here, the group is made up of threads running on different computers, rather than people, and the initial disagreement comes from different threads calling `tryInitialize` with different values. The final agreement comes from all threads receiving the same value from `finalValue`. Somewhere in between, the threads worked it out between each other; that was how they reached consensus. Hence we call the algorithm a consensus algorithm.

If you check our write-once-based solutions from the previous heading, you'll see all this in action. Both examples start with different threads in disagreement (different values are being passed to `tryInitialize`), and we end up with all threads in agreement (we forget what value we passed to `tryInitialize` and just trust the value returned by `finalValue`). Somewhere between `tryInitialize` and `finalValue`, the disagreement was resolved (arbitrarily, by picking the first `tryInitialize` call), and the final result was total agreement across all threads &mdash; consensus.

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

What hidden assumptions exist behind this interface? In other words, what rules govern the relationship between what gets passed to `tryInitialize` and what is returned by `finalValue`? What expected behaviors, if violated, would break the example solutions we built on top of this interface?  Any such rule must be a property of a consensus algorithm.

Here's one I can see: since this is a write-once variable, its value must never change. So all futures returned by `finalValue()` must always resolve to the same value. Different threads must not get different values from `finalValue()`, nor can the final change if a thread calls that method multiple times. Think of the pandemonium that would ensue if we failed to uphold this rule, and in doing so we let the customer cancel an already-shipped order after the box was already on the truck!

In papers and textbooks, this rule is sometimes called **agreement**:

> **Agreement**: If the consensus algorithm returns a value, then no other value has ever been or will ever be chosen. That is: if any call to `finalValue()` returns a value, no other call to `finalValue()` has ever or will ever return any other value.

Here's another rule I can see: the value returned by `finalValue` has to be the value somebody previously passed to `tryInitialize`. The algorithm can’t go rogue and choose something else. For example, an execution like this should not be allowed:

* `WriteOnce<Integer>` created
* Thread 1 calls `tryInitialize()` passing in a value of 1
* Thread 2 calls `tryInitialize()` passing in a value of 2
* Thread 3 calls `finalValue()`, which returns 4 [*](https://xkcd.com/221/)

That might seem obvious, but it's still worth writing down. In *the literature*, this rule is sometimes called **validity**:

> **Validity**: The value chosen by a consensus algorithm must be a value that was previously proposed. That is: the value returned by `finalValue()` must be a value previously passed to `tryInitialize()`. 

One final rule: once `tryInitialize()` has been called, the algorithm gets a reasonable amount of time to do its processing before `finalValue()` futures finally resolve to the chosen value. We can't wait on the future forever. This rule is called **Termination**:

> **Termination**: The algorithm eventually chooses some value; after `tryInitialize()` has been called, the futures returned by `finalValue()` eventually resolve

Agreement, Validity, Termination; seems like a pretty good starting set. However, I think we must still be missing a rule.

By the three rules above, `DistributedWriteOnce<T>` is a valid implementation of a distributed consensus algorithm. But consensus algorithms are supposed to be incomprehensible to mere mortals; I am a mere mortal, and I comprehend `DistributedWriteOnce<T>` just fine. There must be another thing that consensus algorithms do, that `DistributedWriteOnce<T>` does not. And that thing must be a curveball that completely wrecks the strategies we have used so far.

## The Curveball: Fault Tolerance

Earlier, we said the main complexity that distributing an algorithm adds on top of multithreading is that threads in a distributed system must communicate through a network, since they can't share variables. When we said that, I glossed over a really important problem that introduces: networked communication can fail, in ways variable-sharing cannot.

Distribution introduces the problems of **faults** in the system. Hardware can lose power, software can crash, and networks can degrade or disconnect. Technically these problems also affect single-node software, but single-node software can largely ignore faults: if the machine crashes, for example, then the code isn’t running anymore and has no hope of dealing with the situation. This means it can safely a be designed to assume machines crashes never happen in the first place. In a distributed system, a fault can affect some nodes while leaving others online to deal with the consequences.

Distributed system code that keeps working even if some nodes fault is said to be **fault tolerant**. The distributed consensus algorithm we previously implemented, `DistributedWriteOnce<T>` is not fault tolerant: it relies on a single coordinator node and cannot recover if a single fault happens to take down that coordinator node. This is because, in normal operation, all non-coordinator nodes contact the coordinator any time they need to get or set the variable:

```java
public void tryInitialize(T value) {
  return remoteCall(coordinatorId, "tryInitialize");
}

public Future<T> finalValue() {
  return remoteCall(coordinatorId, "finalValue");
}
```

There is no provision here to ever use any node other than the coordinator, so if the coordinator crashes, loses power or otherwise goes away, this code will continue to send messages to the remote coordinator and wait for a response. But since the coordinator is gone, no response ever comes, so the remote calls hang, and thus the algorithm fails to make progress. Since it took just one fault &mdash; one downed node &mdash; to make the `DistributedWriteOnce` algorithm fail to Terminate, the algorithm is not fault tolerant.

Now, the lack of fault tolerance is certainly a limitation, but depending on your use case that may not be a problem. Plenty of systems run on a small number of nodes, and if you have a small network with just a few machines, hardware and software is reliable enough that you won't see faults in the system very often. Rare hardware and software faults only become a consistent nuisance if you have a large enough system to start hitting rare problems frequently. Also, many systems do not need to be highly available: you can take them offline from time to time to do regular maintenance like upgrading software, replacing old hardware, and so on.

If you're dealing with a small system that admits maintenance windows, you can probably get away with something like `DistributedWriteOnce<T>`, which is pretty nice: `DistributedWriteOnce<T>` is much simpler than anything else we're about to discuss, so really you'd be quitting while you're ahead. However, if you have a really big system, or you really can't accept any downtime at all (e.g. maybe you're running the city's E911 or something), then you require fault tolerance, and `DistributedWriteOnce<T>` is too simplistic to meet your needs.

Most of the time, when people talk about consensus algorithms, they mean fault tolerant consensus algorithms. So we are going to worry about making our consensus algorithms fault-tolerant too. Thus, on top of our Agreement, Validity and Termination rules, we'll add a fourth:

> **Fault Tolerance**: The presence of faults in hardware, software or the network do not cause the algorithm to violate the other properties (Agreement, Validity or Termination)

There is some wiggle room baked into this definition. Obviously there is nothing the software can do if every machine in the network loses power simultaneously; typically you either give a constant number of node faults your system can tolerate (typically between 1 and 3) or you say some percentage of machines are allowed to fail ("less than half" is pretty common).

## Implementing Fault-Tolerant Consensus

Our previous strategy of implementing distributed consensus was not fault tolerant because it relied on a single coordinator node to be solely responsible for storing and managing the varible. If we want something fault tolerant, we will have to do away with coordinators and come up with a peer-to-peer algorithm. Every node needs to have its own copy of the variable, and we need to come up with some protocol for them all to come into agreement as to what value they should all store.

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

Each node votes once and sends the same votes to all peers, which means each peer ends up with the same tally. Only one value can reach a majority, and since all nodes have the same tally, they all agree on the value which reached a majority.

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

One thing we could do is add the notion of a "do-over" to our algorithm: if we get to the end of voting and determine it's a split vote, just start over and hold a new vote. We might have more luck and reach a majority the next time around. This is not a bad idea, but I want to avoid it if possible for a few reasons. First, having restarts in a consensus algorithm requires great care: if the algorithm does manage to reach consensus, then restarts and reaches consensus again with a different answer, we end up violating the Agreement property. Second, retries are a lame way to deal with non-termination: retrying doesn’t guarantee termination, and we can't even put a reasonable bound on how many times we will have to retry before we can terminate.

And anyways, there’s a better way to deal with split votes that addresses both of these problems: add a **tiebreaker**. We can define a deterministic rule that chooses a winning value arbitrarily, and run it once all votes are in. Every node has the same tally of votes, so if every node runs the same deterministic rule over that tally to pick a winner, every node will pick the same winner.

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
| Red             | 10              | Yes     |
| Blue            | 10              | Yes     |
| Green           | 5               | No      |

</div>

Here we have two winning candidates by plurality voting rules; to narrow it down further, we would need another tiebreaking rule (a tiebreaker for the tiebreaker). But, whatever rule we use as the tiebreaker-of-tiebreakers would probably work as a tiebreaker in its own right. So let's not use plurality voting.

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

Running the tiebreaker before all votes are in is pretty dangerous: if more votes arrive after we run the tiebreaker, they can change the algorithm's answer and get everything all confused. For example, we could run the bully algorithm at a time where the tally only has votes for `Red` and `Blue`, and decide our answer is `Blue`; but then if other votes come in and even one of those votes picks `Green`, the next node to run its own tiebreaker may run the bully algorithm over `Red`, `Blue` and `Green` and pick `Green`. Then we have disagreement, the algorithm is broken.

Maybe we could work around this by having a different, even more clever tiebreaker than the one we already have. But I doubt it. Even if we had a tiebreaker that somehow couldn't change its mind, we'd still have to deal with the possibility that some candidate value reaches a majority after some node already ran the tiebreaker. If that happens, any node that ran this clever tiebreaker still might end up disagreeing with nodes that saw the majority value and terminated that way. So the algorithm is again broken.

What we do as the tiebreaker doesn’t matter very much; what we really need to do is adjust the trigger condition for running the tiebreaker in the first place. We want the tiebreaker to run after all healthy nodes' votes are in, no matter how long that takes, but we should not wait for votes from failed nodes whose votes will never arrive. But in practice, how do we write that trigger logic?

In literature, a piece of code that can tell the difference between "the remote node is still running, but it's not done yet" and "the remote node has failed, and will never do its work" is called a **failure detector**. Quite a bit of theory has been built around what we can do if we have a reliable failure detector, but the fact that remains, failure detectors don't exist in the real world. (At least, not 100% accurate failure detectors, which is what we need here.) Sure, we can detect that network requests we have sent to other nodes have timed out, but we can't tell the difference between "the reply hasn't arrived yet" vs "the reply is never coming." No matter how long we've waited so far, it’s still possible the reply could arrive if we just wait a little longer.

Without a failure detector, there's no way fault-tolerant way to run a tiebreaker safely: finding the right time to run the tiebreaker would by definition be finding a perfect fault detector. So running the tiebreaker before all votes are in simply isn't an option. On the other hand, waiting until all votes are in isn't an option either &mdash; we would sacrifice fault tolerance. So tiebreaking rules are completely out.

But then, how do we fix our voting algorithm?

## Finding a Lateral Move

Our basic majority voting algorithm doesn't always make a decision if we run it without a tiebreaker. But we also can’t detect when it’s time to run the tiebreaker: either we sometimes wait for nodes that have failed and thus the algorithm gets stuck, or sometimes we don’t wait for nodes that haven’t failed and the algorithm is broken. Tiebreaking is a dead end; we need a lateral move. Can we fix split votes without a tiebreaker?

Well, early on we passed on the idea of handling split votes by restarting the algorithm. Would that work? Well, once again, we have to decide when to trigger the revote. If we wait for all votes before calling for a revote, then we can't sustain the loss of even one node; but if we don't wait for all votes, then the first round of voting could produce a majority after the second voting around has started, resulting in inconsistency. In other words, restarting with a revote has the same problem as the other tiebreaking rules we tried before: without a failure detector, we don’t know when to do it. I guess in retrospect, restarting the algorithm is itself just another tiebreaking rule.









TODO

* What if we tried to use timeouts to rule out bad nodes? That's more clever but it still doesn't work: the node that did the last vote might know it has reached consensus and resolved `finalValue()` futures even though it can't communicate out to anybody else, which is another but more subtle agreement violation.
* Both of these are good setup because the final Paxos solution is to make it safe to restart at any time. The point of synod is to guarantee, at any time, if someone successfully restarts the vote after consensus has been reached, always they will reach consensus again on the same value.
* This is also potentially a good segue out: observe everything else we do looks an awful lot like a tiebreaker. Proving this would be huge: it would show that we're actually overconstrained.

# Intermission

We set out with the goal of learning the problem space of consensus, so that we could see Paxos as the simplest possible solution to an unintuitive problem, rather than an unintuitive solution to a seemingly simple problem. Since then, we've gained an appreciation for why consensus is uniquitious in distributed systems, we nailed down the exact properties a consensus algorithm should have, and we've written quite a few consensus algorithms ourselves, between the `WriteOnce` implementation exercises in Part 1 and the voting algorithms in Part 2. Unfortunately, we have yet to invent a fault-tolerant distributed consensus algorithm, and as of yet we don't have any ideas to try next.

This is the low point in our journey. We understand the problem and we appreciate its difficulty, but we don't yet have a way forward. Things will only get better from here. We're about to turn our attention to the FLP result, which explains what we have been doing wrong so far, and in doing so we'll learn what we need to change about our approach in order to arrive at the Paxos algorithm. Pour yourself your stimulant drink of choice and let's keep going!

# Part 3: The FLP Result

7/17/25: one more observation I’d like to work in below: the situation with the two inbound messages that determine the outcome of the algorithm is unsolvable because it’s an instance of the CAP theorem: if you get partitioned from that node you can either be consistent with its answer (required by agreement) or you can be available (required by termination), but not both.

So the basic flow of this section might be like

* Explain FLP revolves around this situation where the delivery order of two messages inbound to one machine determines the outcome of the algorithm. One order results in one outcome, the other results in the other. Work examples to show we have hit this situation multiple times already
* Preview the basic proof: this situation is both unavoidable and unfixable
* Unavoidable proof: the cute non-termination argument. At any point you can stop the algorithm and for any two messages, there is some delivery order where the algorithm remains undecided, so you don’t have termination. A simple way to say this might be (a) by definition of not terminated both red and blue are still possible outcomes, and (b) for any pair of messages the only outcomes are (decided red) (decided blue) or (did not decide). More cases: if both outcomes are (decided blue) or both are (decided red) you have already made a decision, so this isn’t a possibility. And if you have (decided red) vs (decided blue), that’s the “FLP situation” we were avoiding, so that’s not possible. Thus the only possible set of outcomes is (decided red/blue) vs (remains undecided) or both (remains undecided). But if every two messages have at least one delivery order where you remain undecided then you must not guarantee termination. The beginning of this proof must assume the “easy” case of no faults or crashes or dropped messages, which nicely removes cases of messages being lost. Certainly a fault tolerant algorithm would need to work in a non-faulty network.
* A different way to format same proof: every algorithm that guarantees termination must eventually get to the point where there is no option to remain undecided, ie a state where we do not have a decision, but no matter what message is delivered next a decision will have been reached. What decides is the delivery order. The order between nodes doesn’t exist, delivery to two different nodes is effectively parallel, so we have to rely on the order of two messages for a single node. But then we have two messages inbound to the same node where one order is red and the other is blue, that’s the situation we tried to avoid. So termination leads us into the FLP situation directly.
* Unfixable proof: preview for readers who know the CAP theorem that this is CAP, per above
* Consistency leg of proof: if you wait to hear from the other guy, you aren’t available if he crashes. Non-termination
* Availability leg of proof: if you move on, different nodes can end up with different answers, non-consistent / agreement violated
* Maybe mention failure detection: if you can modify P so that you can tell the difference between alive and dead, you can decide on the fly whether to be CP with the live node or AP with the dead one. But alas, perfect failure detectors don’t actually exist
* Finally set up the lateral move: so we definitely can’t solve the CAP theorem but we can avoid the situation by making the algorithm that doesn’t terminate as per the unavoidability proof above. The result can always keep wasting time without a decision, but maybe we can make continuing to remain undecided progressively more unlikely over time, and end up with a probabilistic guarantee. Note that FLP said this in the original paper
* The majority voting algorithm can be adjusted into something that avoids the FLP situation. The simplest way to do it results in a classic algorithm called paxos.

--

TODO:: a good cold open might be a repeat of Lemma (2? 3?) the inscrutable one. "Refuses to elaborate, leave"

TODO: open as the proof generalizes what you already saw with the tie breaker / pick your poison

---

2/6: as a setup to the below, it may be worthwhile to frame the way we abstracted the algorithm

Consensus algorithms appear to be nondeterministic. When you have competing proposals, the algorithm picks one arbitrarily / largely at random. Different executions starting from the same starting conditions can end up deciding on different answers. They’re non-deterministic.

That’s odd though .. look again, all of our code is 100% deterministic so far. No random calls, no timeouts ... where is the non-determinism coming from?

The answer is the network.

Different network delivery orders cause a deterministic algorithm to potentially decide different things in different executions from the same starting conditions.  

That suggests we should pay close attention to network messages and the order they are received and processed, and how that affects what the deterministic code we write is doing. Indeed, that is the way FLP looked at it too.

1/30: came up with a new option for flowing this section

First, explain the key situation that FLP is concerned with: there are two messages addressed to the same node, both have already been sent, and the relative order those messages are processed determines what the algorithm ultimately decides.

That probably sounds abstract, so show it again in terms of the split vote situation: 5 nodes, 2v2 split vote, two inbound proposal messages for both values to the last node. Now we have two inbound messages and the relative order they are processed decides for the whole system (out rule is first in wins, but there could be others)

Preview now we have to cover two topics:

* Why this is inevitable
* Why this is pathological

Why inevitable: 

First, point out we start out bivalent (don’t use that word). At the start of the algorithm we can’t do anything because nothing has been proposed. In one possible future, only red is proposed and by validity we eventually decide red. In another, only blue is proposed and we eventually decide blue. Thus both outcomes are still possible at the very beginning.

But of course we need to eventually get to the point where only one outcome is possible.

Surprising claim: there must be a point in the algorithm where both outcomes are still possible, but there is some message which has already been sent, and once that message is processed, we’ll have picked an outcome.

To unpack: that message either arrives after an outcome has been decided and thus does not affect the outcome, or it arrives before an outcome has been decided and causes a decision to be made. But there is no way that message can be processed and end up with the system still undecided.

Quick check: this exists in the voting algorithm. 

But how do we know it always happens? Because if that never happens, there’s no termination guarantee; the algorithm is allowed to kick the can down the road forever. Induction proof.

Okay, so there is some message, it has already been sent, and once it is delivered, no matter when it is delivered, we can still be undecided.

But can how we be undecided still? Clearly sometimes the message arrives and we choose blue, other times it arrives and we choose red. What affects the decision? It can’t be something about the message itself -- it has already been sent! The only thing that can affect it is the state of the receiving node, which can only be changed by some other message.

So there must be some second message now, which has the ability to affect the outcome. If that other message arrives first, then we decide one way, or if it arrives second we go the other way.

And so boom, no matter what the algorithm is, we have a situation where sometimes the delivery order of two inbound messages for the same node decides what the algorithm decides. It is inevitable!

But why pathological? 

We’ve already kind of seen it.

In this situation where one node’s delivery order makes the global decisions, what should the other nodes be doing?

Waiting? Well then if the node crashes before receiving either message, we’re stuck. The algorithm hangs. We can’t just wait on the node.

So we should make a decision without that node? Well, that node isn’t guaranteed to crash ... so any decision made without that node’s help must be consistent with what the node decided. But it’s literally impossible to know in advance what the node will decide: the decision is determined by the order those two messages are delivered, and nobody else knows the delivery order in advance!

Now if we could tell the difference between crashed and still up, then we could do it. In the case where the node is guaranteed crashed, we can determine nobody saw the decision and then make a decision without that nodes help; in the case where the node hadn’t crashed, we keep waiting. But sadly, we don’t have fault detectors in the real world.

Summarize

Segue out: well, okay, now we know we’re overconstrained, just picking the properties we picked means we inevitably get into this situation where one node’s message delivery order ends up being critical, and there’s no way to safely recover from losing that node before the delivery order is discovered.

So we’re stuck and consensus is impossible? Well obviously consensus algorithms exist, so it’s not impossible. But do you see the lateral move? How do we get out of here?

Well, one reason we said it’s inevitable is because the algorithm needs to guarantee termination. Does it need to guarantee termination?

I mean, guaranteed non-termination would be pretty bad. But non-guaranteed termination might not be so bad...

In fact, (pull the quote from the paper)

So that’s how we’ll do it: we need to avoid the situation where two messages inbound to the same node ultimately decide the outcome, and we’ll have to sacrifice termination guarantees to do it.

Now the segue into Paxos is like:

In other words (example of the above), we will need to avoid split votes, and we will end up sacrificing termination guarantees (introducing a potentially infinite loop) to do it.

Can we do that? Well the only reason split votes are a problem, per FLP, is because two different proposals are inbound to the same node. What if we only vote on one proposal at a time? That way the decision isn’t, A vs B, it’s A vs start a new vote.

Of course we have to deal with the fact that two proposals might start simultaneously. We can organize these into rounds, so we keep voting in rounds until we make a decision. In each round, the decision is between A or “move to the next round.”

Since every round we can keep going, we have no guarantee of termination, as FLP predicted. But if we have a good probability of coming into a decision in one round, then have a really good chance of doing it in two, a really really good chance of doing it in three, and so on. So we have non-guaranteed termination rather than guaranteed non-termination.

Network programming offers a class of collision avoidance algorithms, any of which can help avoid proposals from stomping on one another, driving up that probability of terminating per round. In the end, the number of rounds is pretty manageable, and usually one!

But even then we still have a problem: how do we safely change rounds? We have solved the problem of “I don’t know whether that machine decided A or B,” but now we have the problem “I don’t know whether that machine decided A or moved to the next round.” But that’s fine! Because there’s a decision that’s safe either way: assume conservatively that it did choose A.

So in the next round, propose A again. If we were right and A had been chosen, voting for A again is wasteful, but isn’t a correctness problem. If we were wrong, A is still a valid choice (someone must have proposed it for it to appear in a prior round), so now we have terminated by choosing a valid option. Either way, we’re happy!

This I think gets us to the interlock idea, where you check what the most recent voted value is and help that one, on the assumption that one was picked. You know all earlier round values that received votes did not reach majority because the coordinator of the next round never saw it during its first round majority lock and read. The only one you’re not sure about is the most recent round you saw that received votes. 






---

Part 3 is FLP

* The proof thinks a lot about termination. What does it look like when a consensus algorithm makes a decision?
* We need a way to talk about algorithms generically. Observation: consensus algorithms we have built so far are all snippets of 100% deterministic code which still decide nondeterministically. The nondeterminism is network delivery order
* Step model: think about how a deterministic algorithm interacts with a nondeterministic network. The network delivers a message, some deterministic logic runs, maybe we send some more network messages too. Call that logic a step
* How do steps relate to decisions? Well, first of all a consensus algorithm must never change its mind once it makes a decision. So there should be one step that makes the decision aka terminates the algorithm: before this step, it was possible to go either way, but afterward every node is fated to return the same finalValue
* Thats interesting, because if you think about it, every step runs on one node. So in the end, in every consensus algorithm termination, one node does a thing which terminates the algorithm
* Hmmmm what if we lose that node, and the step goes away?
* Actually, frequently losing a step that would have made a decision is not a problem. Consider a 5 node network where there are already two votes for blue. If we lose a node that was about to decide blue, but then some other node comes and decides blue anyways, no biggie. Frequently we can replace the lost step with some other step
* But Termination doesn’t mean we can decide, it means we guarantee some decisions eventually made
* So we’re not interested in the step that decides; we want to rewind the clock a little bit.
* If the algorithm guarantees a decision, we should eventually get to a state where the algorithm could still decide either way, but some step has been triggered (its network message has been sent but not yet received), which guarantees a decision is made no matter what it does 
* That’s a lot to unpack. Do examples of previous algorithms to show what this decision-ensuring step looks like and what the vulnerable point of the algorithm is where we have sent but not received that message
* Why must a Terminating algorithm have such a step? If it didn’t, then at every state of the algorithm, there’s some network delivery order for which we don’t terminate, basically by definition.
* Okay, so a Terminating algorithm will eventually trigger a decision ensuring step. Alas, that is a big problem for us. 
* In this situation, there is some execution where the node receiving that message and executing that step ends up deciding all by itself, using local information only it has
* Proof step 1: the step can decide either way, because the system can decide either way. If the step always picked zero, for example, that means if the system decides 1 and then the step executes, it would change the answer to 0, which is illegal
* Proof step 2: so what changes which way the step decides? Well the trigger message has already been sent and can’t be changed; so at this point the only thing that can change what the step decides is local state on that node. And the only thing that can change the nodes state is other steps running on that node
* So now we know there’s a step, or set of steps which run on the node, which change what our magic step decides. If the magic step goes first, we decide one way (eg 0), but if the other steps go before the magic step we decide 1 instead
* Example of split vote illustrates this. 
* But basically now we have a node making the decision unilaterally using information only it has. What if the node crashes?
* Well, we could do nothing and stop. But then we do not terminate, which means we are not fault tolerant
* We could also try to keep going and make a decision anyways. But here’s the problem: if we keep going, we have to be able to always keep going, even if that node doesn’t crash. Because we don’t have a failure detector: no matter how long we have waited for that node to make up its mind, it’s still possible it’ll respond if we just wait a little longer
* Thus requirements for keeping going: we must make a decision, in case p has crashed; we must decide 0 in case p has decided 0; we must decide 1 in case p has decided 1. That doesn’t make sense. Ergo the code to keep going actually doesn’t exist.
* But if we don’t keep going, then we just get stuck if that node crashes. Either way, we are not fault tolerant
* So that’s the X-factor and that’s how it prevents us from being fault tolerant.
* So let’s just design an algorithm that avoids this case!
* Not so fast, say FL&P: any algorithm that avoids this case doesn’t guarantee termination even if there are zero faults!
* To avoid this case, we just guarantee, up until we happen to make a decision, every step that has been triggered has an “out,” some ordering of network messages such that, once that step has executed, the system is still undecided. Otherwise, the opposite is a step that ensures a decision, which is the thing we must avoid
* But if every step has an out, then no matter how long you have been executing, there is still a chance you happen to randomly take that path. The result: an algorithm with at least one execution (potentially infinitely long!) which does not terminate.
* So pick your poison, you can’t have it all:
* Your algorithm does not guarantee termination no matter how many faults
* Or your algorithm is guaranteed non-termination in case of just one crash, like DWO, basic voting and tiebreaker at the end
* Or your algorithm contains fundamentally broken logic that corrupts the decision state because you don’t have a failure detector, like tiebreaker before the end
* Which would you pick?
* Well, non-guaranteed termination is not the same thing as guaranteed non-termination
* Can we avoid the X-factor and end up with an algorithm that never terminates, but be ensured that the probability of not terminating is always falling geometrically 
* Indeed we can; one such way of modifying our voting algorithm into that is called Paxos.

How do we do that? We need a voting algorithm with an infinite number of rounds since we need to keep trying to terminate, and also avoids ever having a decision-ensuring step.

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
