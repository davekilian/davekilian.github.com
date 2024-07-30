---
layout: post
title: Consensus, FLP and Paxos
author: Dave
draft: true
---

To understand the algorithms that underpin distributed systems, one must first accept this fundamental truth of all distributed systems: 

> Stuff is broken all the time, and we have no hope of fixing it all. ¯\\\_(ツ)\_/¯

Why? Think about it this way:

How reliable is the device you’re using to read this? I mean, it probably works fine most of the time, but occasionally I’m sure you run into snags: freezes, crashes, overheating, dead batteries, random network disconnects, etc. In distributed systems, these kinds of problems are called **faults**. So how often does your device fault? Would you say it’s on the order hours, days, weeks?

Well, that's just with one device. What if you had to manage two thousand of them? What if you had to keep all of them working all the time?

Let’s say you get hit by one of these little snags once every two weeks. Then we’re going a smidge over 1 million seconds between faults:

<div class="overflows" markdown="block"><center>

$$2 weeks \times 7 \dfrac{days}{week} \times 24 \dfrac{hours}{day} \times 60 \dfrac{minutes}{hour} \times 60 \dfrac{seconds}{minute} = 1,209,600 seconds$$

</center></div>

Not too bad. But now let’s say we have 2,000 devices to manage. We’re going to start seeing random device faults about 2,000 times more often, reducing the average time between faults to:

$$1,209,600 \div 2,000 = 604.8 \; seconds$$

That's one new fault every 10 minutes . . . 24 hours a day, 7 days a week, forever. See the problem? Even if we do ever manage to get on top of all the weird stuff going on in our network, we'll never be done for more than 10 minutes at a time. 

The random crashes, freezes and disconnects that didn't seem like a big deal before are insurmountable at scale. Cloud providers have this problem times 100: they operate huge networks with many thousands of computers distributed across the world. Every minute, there’s new nonsense cropping up somewhere in the network; it happens so much because the network is so large. They can spend as much money as they want and hire as many people as they like, and still never get ahead of all the problems constantly starting up.

So that’s that: the hardware running your code is not 100% reliable, the network is not 100% reliable, operating systems are not 100% reliable, and we have no path for to 100% reliability for any of these things. Does that sound terrible? Because distributed systems people know all this, and they’re pretty zen about it.

[ this is fine dog meme ]

That’s because we’ve figured out how to write software that papers over these reliability problems. Today, large systems everywhere are underpinned by **fault-tolerant** software algorithms, which work even when the infrastructure they run on doesn’t. 

But how can code work when the computer running it doesn’t? There’s no magic here; fault tolerant software works because systems fail in predictable, manageable ways. Any time your code asks the system to do something, one of three basic things will happen:

* The system does what you asked it to do
* It does what you asked, but it takes a *reeeally* long time to do it
* The thing you asked for just never happens at all

However, it’s generally safe to assume the system will not go rogue and start doing random things you didn’t ask it to. All the things that will happen are things you coded to happen ... you just can’t be sure how soon anything will happen, if ever. 

It’s also safe to assume faults aren’t very widespread: obviously if all our computers have crashed and there is nothing left to run our code, then there’s nothing our code can do about it; but if just a few machines are having problems, the other computers can compensate via two basic strategies:

* **Keep backup copies** of all data, in case a machine storing it crashes
* **Keep retrying things** until they happen, to deal with delays and dropped requests

In this post we’re going to take the first baby step on our journey writing fault tolerant code: we are going to reinvent the concept of variables for the world of fault-tolerant programs.

## Fault Tolerant Variables

Say we have a network of computers. Call each computer a **node**. We might picture the network of nodes like this:

DIAGRAM

Neither the nodes nor the network are completely reliable. At a low rate, the network randomly delays messages, and sometimes drops them so they are never delivered at all. The nodes also crash sometimes.

Can we implement a fault tolerant variable in software for this network? By this I mean I want a software primitive with the following properties:

* There is one variable
* Any node can get it
* Any node can set it

. . . and all of these properties hold even if the network is having problems, or some nodes have crashed. The variables are supposed to be fault tolerant, after all.

DIAGRAM: nodes a network, thought bubble question mark in the middle for a variable

Oftentimes the hardest part of a problem is figuring out where to start. Let’s attack it this way: we’ll start with a simple approach we know doesn’t work, and then try to fix it into working. Maybe that’ll yield a good solution, or maybe along the way we’ll learn something important about the problem.

## A Simple (But Wrong) Approach
 
We already know how to make regular, non-distributed variables, so let’s make a plain old variable on one of our nodes, and set up an RPC server on that node so the other nodes can get and set the variable remotely.

DIAGRAM

The node we picked to store the variable is now special; let's call it the **leader**. For the leader, the variable is just a plain old local variable like any other. The other nodes, which we'll call **followers**, access the variable by sending RPC messages to the leader.

The leader's logic will roughly be . . .

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

The followers:

```
client {
  get {
    return leader.rpc(get);
  }
  
  set(value) {
    leader.rpc(set, value);
  }
}
```

Now we have one variable any node can get or set. That was easy! 

. . . too easy.

What happens to our algorithm if a node crashes?

DIAGRAM: another copy of the RPC diagram, last one in the previous section

If a random node crashes, most likely that node will be a follower, since there are so many followers and only one leader. If a follower goes offline, the variable should be safe and sound: it's stored on the leader, which is still online, and all the other followers are still in contact with the leader:

DIAGRAM: same diagram with a random follower Xed out

But the leader is not immune to problems. If a random node crashes, it very well could be the leader. So what if it's the leader that goes down?

DIAGRAM: same diagram, but with the leader Xed out

Now we have a problem. With the leader gone, so is the variable. All the follower nodes are still up and running, but they're only programmed to send RPCs to the leader, and the leader isn’t going to respond now that it’s offline. Our variable has vanished along with the leader; and since it only took one fault (the leader crashing) to do it, we have to accept our variable is not fault tolerant.

That’s fine though, this was the plan all along: now that we have a basic sketch of a design, we just need to figure out how to fix it into something fault tolerant. Let’s give it a shot.

## Broadcast Replication

The single-leader design didn’t work because there is no safe quarter for our variable: no matter what node we put the variable on, it's possible we could lose that node, and the variable with it. The only way to definitely survive one node crash is to have at least two copies of the variable on different nodes. That way, if any one node crashes,  no matter which node, we’ll still have one live backup copy we can switch over to. But what if two nodes crash? Or three?

Heck, to be maximally safe, let's just put a copy of the variable on every node:

DIAGRAM

Every copy of the variable is called a **replica**. The process of creating and updating replicas is called **replication**.

In this new setup, getting a variable is simple: every node already has its own local replica of the variable, so to get the variable, just read to your local replica like any other variable. To set the variable, let's use a **broadcast** protocol: the node that wants to update the variable sends a "set" RPC to every other node, and each other node in turn updates its local replica to reflect the update it just received:

DIAGRAM

One good thing I can say about this new design: it’s definitely fault tolerant. Each node has its own replica of the variable, so as long as we have at least one live node which has not faulted, we also have a live replica of the variable. All remaining nodes can still send set RPCs to one another, so the variable keeps working even if some nodes crash. It is definitely fault tolerant.

Unfortunately, that’s just about the only good thing I can say about this idea. This design is majorly flawed.

Even with fast computers and fast networks, it still takes some amount of time for a broadcast to reach every node. What if two nodes happen to do an update simultaneously?

DIAGRAM

Both sets of RPCs race, so it’s possible for different nodes to see the set messages in different orders:

DIAGRAM

If the updates are processed in different orders, then different nodes will end up with different values for their respective replicas:

DIAGRAM

This means the different nodes disagree what the current value of the variable is. That’s pretty messed up! I’m not sure what kind of software programs you can write on top of a variable that can get all confused like this. We’re going to have to find a way to keep them in sync.

However, keeping replicas of a variable in a sync turns out to be a very hard problem. So tricky, in fact, that we probably don’t want to tackle it right away. Let’s choose a slightly easier problem: how can get the replicas in sync just once?

If we can solve that problem, and apply it to this broadcasting replication algorithm, the result will be a kind of “write-once” variable: one that starts out null, can be set once, and from then on is immutable. To implement that, we just start out with all replicas initialized to null, and then anytime someone wants to set the variable, we run this “get replicas in sync just once” algorithm to decide what the permanent value for the variable will be, and stick with that forever. A real implementation of fault tolerant variables will of course need to support any number of set() calls, but maybe once we’ve figured out write-once variables we can extend the idea into many-write variables.

So for now, let’s focus on the next step: if multiple nodes try to set the variable at the same time, how do we get the replicas to get in sync just once?

This question has a name. It’s called **consensus**.


## Consensus 

Consensus is the problem of getting a group of nodes to agree &mdash; just once! &mdash; on the value of some variable. In our case, the value we're trying to agree upon is what value our fault tolerant write-once variable should be set to. This is easy if only one node calls set(); it’s harder if multiple nodes call set() simultaneously. 

So, what do we need a consensus algorithm to do?

To start, the basic reason we want a consensus algorithm is to ensure all nodes agree on the value of the variable by the time the algorithm finishes. In “the literature,” this most basic property is sometimes called **Agreement**. When we say all nodes should agree "by the time the algorithm finishes," we're also assuming the algorithm should finish in the first place. That is called **Termination**.

If you like technicalities, there technically is a way to achieve Agreement and Termination without really solving the problem: you just hardcode an answer. For example, if you're trying to write a consensus algorithm for integers, "always return 4" is technically a valid consensus algorithm according to our definition so far: all nodes will agree the value is 4, and the algorithm will terminate very quickly. But algorithms like this are useless for the fault tolerant variable problem; we'd end up with a fault tolerant constant instead!

We want the agreed-upon value to be the one somebody wanted. If only one node calls set(), the variable should be set to that value. If two nodes call set() at the same time with different values, the algorithm should pick one of those two values. If both set() calls specify the same value, the algorithm should always pick that value. This idea is called **Integrity**.

And, of course, we can't forget the uber-goal of ending up with something **Fault Tolerant**. 

Then it’s settled: we believe a consensus algorithm should provide:

> **Termination**: The algorithm exits.
>
> **Agreement**: When the algorithm exits, all replicas agree on a value.
>
> **Integrity**: That value is one somebody proposed.
>
> **Fault Tolerance**: A single fault cannot violate any of the properties above.

Let's try to invent an algorithm that accomplishes all of this.

Do you know of any real-life algorithms for a group of people to come to an agreement?

For example, think of a group of friends that want to out to eat somewhere. To do that, first they need to pick where to go; that's an agreement problem. What might happen next? Maybe someone throws out an idea, someone throws out another idea, some people agree, some disagree, eventually a group opinion starts to form. The tide starts to turn when someone finally says

<center>"I vote we (blah blah blah . . .)"</center>

Oh yeah, voting! Majority-rules voting is an algorithm that results in a group agreement. Maybe we could code up something like that? Let’s try it.

## Majority-Rules Voting

Let's code up an algorithm where nodes throw out proposals and vote on them, just like in the restaurant example above. However, unlike messy real life where people have preferences, we'll make it so each node has no preference whatsoever: each node will vote for whichever option it heard about first, and never change its mind.

To keep things as simpile as possible (for now), let's only allow two values to be proposed. Since I’m American, we'll call those options <span style="color:red">red</span> and <span style="color:blue">blue</span>; but these options can stand in for anything: 0 and 1, yes and no, apples and oranges, etc. Supporting only two options is too simplistic to implement a real consensus algorithm, but in computing you usually end up being able to have exactly zero of something, exactly one of something, or any number of something. If we can figure out how to have exactly two options, there's probably a way to extend it from 2 to $N$​ options.

With that, let's start by having each node track its own vote, initially null:

DIAGRAM

To get things going, some client of our system needs to call set() &mdash; a.k.a. throw out a proposal. We'll implement that by broadcasting a message to all other nodes, telling them all to vote for a specific proposed value, which can either be <span style="color:red">red</span> or <span style="color:blue">blue</span>.

DIAGRAM one proposal

Then we'll have each node vote for the first proposal it hears about. Since there is only one proposal in this example, every node hears about the same proposal first, so they all end up in agreement immediately:

DIAGRAM all nodes colored in the same color

Lucky us! We won't always be so lucky though. There's no central coordination involved in creating proposals, so we have to anticipate multiple proposals could be thrown out at the same time. If we have multiple racing proposals, they might agree, but they also might not:

DIAGRAM three proposals, two blue and one red

With multiple competing proposals, different nodes can receive different proposals first, and end up voting for different things, just like when we were talking about our broadcast replication algorithm before:

DIAGRAM

Now the votes don't agree. But that's okay, we don't need the votes to agree; we just need to figure out which proposal got the most votes. For that, we can have every node tell every other node who they voted for:

DIAGRAM

Once every node knows what the final votes were, they can each independently count the votes to pick the winner. Since it’s the same votes no matter who’s counting, all nodes end up picking the same winner:

DIAGRAM

As long as some value reaches a majority of votes, that value will be the one and only value every node picks as the winner.

Here's the full algorithm, for reference:

```
consensus {
  vote: Red | Blue; // this node's vote
  nodes: Node[]; // all nodes, including self
  
  init {
    vote := null // haven't voted yet
  }
  
  // a caller wants to propose a value
  propose(value) {
    nodes.all.send(proposal, value)
  }
  
  // received a proposal from another node
  on received proposal(value) {
    // accept the first proposal, ignore others
    if (vote == null) {
      vote := value
    }
  }
  
  // a caller wants to read the final value.
  // returns null if decision still in progress.
  get() {
    counts: map{ from proposal to int }
    foreach node in nodes {
      counts.add(node.get_current_value())
    }
    
    // returns null if there is no majority yet
    return get_majority_proposal(counts) 
  }
  
  on received get_current_value() {
    return vote
  }
}
```

Now we have a complete algorithm, that pretty much works:

* **Agreement**: ✅ &mdash; only one value can reach a majority, and all nodes will see that value is the one that reached a majority
* **Integrity**: ✅ &mdash; nodes only vote for a value someone proposed; so the value that got the most votes was proposed by somebody
* **Termination**: . . . oh wait, does this algorithm always terminate?

In our new algorithm, agreement is reached once a value reaches a majority; so we can only terminate if some value reaches a majority. But what if we're unlucky, and we end up with a split vote?

DIAGRAM 3 v 3 split vote

Now no value has reached a majority, and since there are no more nodes left to vote, no value ever will reach a majority. Our algorithm doesn't terminate!

Well, there's a simple workaround for that: just require an odd number of nodes. If there are only two values you can propose, and there's an odd number of nodes, *some* proposal has to have reached a majority once all votes are in!

DIAGRAM

Great, let's assume an odd number of nodes and check again:

* **Agreement**: ✅
* **Integrity**: ✅
* **Termination**: ✅ &mdash; with only two values and an odd number of nodes, some value will have reached a majority once all votes are in
* **Fault Tolerance**: . . . we might have a problem here

In a fault tolerant algorithm, it should be okay for one node to crash. But we just said it was really important for us to have an odd number of nodes . . . if a node crashes, we're left with an even number of nodes again, and split votes become possible again:

DIAGRAM

Uh oh, the wheels are really starting to fall off again, aren't they? Having an odd number of nodes isn't sufficient to protect from split votes; we need to find another solution.

Okay, okay, hear me out. I have another idea:

## Tiebreaks and Takebacks

There's no way to prevent split votes, so there's no way to ensure some value always reaches a majority. Maybe it's not as bad as it sounds. Currently, once all the votes are in, we just pick whichever value reached majority; maybe we can just tweak that rule again in the case where the votes tie.

Let's add this tiebreak rule: if a color has a majority of the votes, we pick that color, but in the event of a tie, we pick red (chosen fairly by asking my 4-year-old his favorite color). Now if a node crashes and leaves us with a split vote, we still get an answer:

DIAGRAM

But . . . no, wait, hold on now, we can't do this. We're making a decision before all the votes are in! Right now we have a tie vote and we picked red, but how do we know that last node is actually offline? What if it's completely healthy, and just happens to be the last node to vote? What if it comes back moments later and votes blue?

DIAGRAM

We changed our mind! At one point in time, we had a 3v3 split vote, and all nodes agreed on red, per the tiebreak rule. But then that last vote came in, and everybody changed their mind to blue. You can't do that! What if somebody already saw that the outcome was red, and someone sees the new outcome is blue? That doesn’t sound like Agreement to me.

Agreement is total; it requires all nodes to always agree on the same value. Once you declare done, you have to stick with that value forever.

Things get worse from here. The problem we just uncovered isn't specific to the tiebreaking rule we chose; it's going to be a problem with any tiebreaking rule. Using a tiebreaker is totally fine *if* the final node is offline and is guaranteed never to come back. But how do we know the final node is offline and not coming back? No matter how long we wait for the last node to enter its vote, it is always still possible for it to do so some time in the future. That means, no matter how much time has passed, it's never safe to run the tiebreaker rule. If it’s never safe to run the tiebreaker rule, we simply can't have one.

So we can't prevent split votes, and we can't resolve split votes either. This is looking an awful lot like a dead end.

Hm.

## Something Has Gone Very Wrong

It would seem we have jumped down a rabbit hole much deeper than we at first imagined.

Lets recap how we got here:

We started with the simple goal of inventing a fault tolerant variable &mdash; kind of the most basic thing you would need if you intend to write fault tolerant software. 

Then, in about 30 seconds, we invented a single-leader algorithm that worked, except it made no attempt whatsoever at fault tolerant. To add fault tolerance, we then added backup copies of our variable. But more replicas meant more leaders, and that lead to a new problem: how do we keep those replicas in sync? That’s how we got started talking about consensus.

We came up with a pretty good base idea for consensus, one even rooted in metaphor for consensus in real life. But rather unexpectedly, it turned out to be a complete dead end.

We just keep layering on the complexity, and still we don’t have a solution in sight. Did we take a misstep somewhere? This seemed so simple at the start; how did we get so stuck?

If we’re looking for consolation, at least we’re in good company. At this stage, a generation of distributed systems researchers put an awful lot of thought into this problem. They were able to answer lots of related questions: things that don't work work, properties any solution must have, different ways of simplifying the problem and then solving the simplified problem. But for years, nobody had an answer to the main question: how does one design a fault-tolerant consensus algorithm?

I would like to invite you to join in the tradition by mulling it over yourself. What is wrong with the approaches we've tried so far? Can we fix them? If not, why not?

If you want a little bit of direction, consider the dead end we just ran into with split votes. We were left with two unacceptable choices:

* Include a tiebreaker, which caused the algorithm to Terminate but does not provide Agreement
* No tiebreaker, which caused the algorithm not to Terminate but does uphold Agreement

Why do Agreement and Termination seem to be at odds with one another?

<!--

Omfg, self nerd snipe: this is CAP.

* No network partition: CA
* Tiebreaker: AP
* No tiebreaker: CP

Funkay.

-->

Think about it! This page will still be here when you get back.

<center>* * *</center>

Welcome back! How did it go? I'm guessing you're still stuck, but don't worry &mdash; we'll sort this all out soon enough.

If you repeatedly find yourself unable to solve a problem, and especially if all your solutions keep hitting the same set of dead ends, the next thing to do is to try proving the problem is impossible to solve in the first place. This is exactly what three researchers managed to do in the mid-1980s. In their paper *Impossibility of Distributed Consensus with One Faulty Process*, Fischer, Lynch and Paterson (the "FLP" in what later became known as the "FLP result") explained exactly why nobody could come up with a fault-tolerant consensus algorithm.




<!--

I have a phone note not copied here yet; I want to do this differently. Focus on “way-ness” where each step of the algorithm is a 0-way, 1-way or 2-way decision. Show that 2-way decisions are DOA, you either have a deadlock or a race condition in your algorithm. Suggest that you find a way to do 1-way voting.


---

Much of the time, big breakthroughs in hard problems come from finding an interesting new way to look at and analyze the problem; FLP is an example of this. Let’s see how they look at consensus algorithms:

## Decisions, Decisions

Consensus is easy when only one value is proposed. The hard part is resolving conflicts: if one node proposes red and the other blue, only one of those can end up being the agreed-upon value. How does an algorithm decide between them?

Forget the mechanics of how the algorithm makes the decision; after all, every algorithm can do so in its own unique and special way. But still, there’s a general, overall shape to how these decisions get made.

Say we’re at the start of a consensus algorithm &mdash; any consensus algorithm &mdash; and say one node is proposing red and another is proposing blue:

DIAGRAM

I claim it is possible for the algorithm to pick either red or blue at this point. This is because the algorithm needs to be fault tolerant, and so it’s possible for either the red proposer or the blue proposer to crash without halting the algorithm. Say the red proposer crashes at this point; then it dies being the only node that ever knew red had been proposed. The remaining nodes only know that blue was proposed, so by Integrity the algorithm must decide blue:

DIAGRAM - red proposer X’ed out, all other nodes blue

But if the blue proposer were instead the one to crash without getting the word out, the rest of the system will only see that red was proposed, and therefore the algorithm must decide red:

DIAGRAM - blue proposer X’ed out, all other nodes red

So at the beginning of the algorithm, before either node crashed, both proposed values were still on the table. So every algorithm starts with every proposed value still in the table.

Let’s start drawing a timeline of the algorithm’s execution:

DIAGRAM - timeline, with “both still possible” pencilled in at the start

What else can we add to the timeline? Well, we know by the end of the algorithm we must reach agreement, so at the end we must have picked either red or blue, and eliminated the other:

DIAGRAM - same timeline, with end pencilled in “one value chosen”

Somewhere in the middle, there must have been a decision, taking us from “all values are still on the table” to “one and only one value has been chosen.” And since the agreement property requires us to never change our minds after deciding upon a value, the decision must be a single step, which starts with multiple values still possible and ends up with a single value having been decided. We can draw this step a single point in time on our timeline:

DIAGRAM complete timeline with start, decision step, end as points which create two line segments labeled “both values possible” and “one value chosen”

Note that every step of a distributed algorithm runs on a single nod, and as a step of the algorithm, the decision step too must happen on one, and only one node. This is interesting, don’t you think? The algorithm must include a critical step, which must happen on exactly one node, yet that node could potentially crash before, during or after that critical step. 

Anyways, we may not have the whole story here, but we learned something interesting:

> In every consensus algorithm, there is a **decision step**, running on a single node, that decides the final value on behalf of the entire system

What else can we say about every consensus algorithm?

## Nondeterministically Yours

By our own analysis, both red and blue are still options at the beginning of any consensus algorithm’s execution. That means two runs of the same algorithm, starting in the same initial state, do not always have the same outcome. So consensus algorithms are **nondeterministic**. All of the algorithms we have drafted so far are indeed nondeterministic. 

What’s weird is, all of our draft algorithms so far consist entirely of deterministic steps. We don’t generate random numbers, start timers or spin up racing threads. So where is the nondeterminism coming from? 

It’s coming from the network. Variances in exact timings and other factors mean that the set of messages being received by a node can arrive in any order. That results in nondeterministic behavior in the overall algorithm.

And since the decision step picks red or blue nondeterministically, the delivery order in network messages must somehow influence what value the decision step chooses &mdash; in every consensus algorithm!

Now we know enough to analyze consensus algorithms the FLP way: we simply ask ourselves

1. What is the decision step?
2. How is it influenced by the order network messages are processed?
3. What happens if a node running the decision step fails?

We can try this out on a few of the example algorithms we already came up with before. Then maybe we can spot the reason we kept running into dead ends.

## Our Examples, Revisited

Let’s play spot the pattern.




## One-Way, Two-Way, Red-Way, Blue-Way


<!--

From my phone notes on 3/22:

1. A lot of the time, insights are from finding a new angle to view a problem
2. Observe that each algorithm has a decision point where we go from bivalent to univalent and never go back. Observe that happens on one node because every step happens on a single node
3. Observe so far our algorithms are all deterministic, yet behave non-deterministically. This is because message delivery order is the source of nondeterminism. 
4. Suggest therefore we look at the problem from the point of view of the decision step and how it relates to nondeterministic message delivery order
5. Now we’re in a good place to show a few diagrams and play spot the pattern
6. The end of this section is a new term: one-way decisions either decide on a specific value or don’t make a decision. Two-way decisions either decide on one value or another. Note all of our problem cases are two-way decisions
7. Revisit the problems from before as two-way decisions. This rehashes lemma 3
8. First: if you have a two-way decision and the node crashes, and you don’t do any further work, then the algorithm is dead. Duh. But think about the case where we do more work
9. Second: whatever steps we include in the algorithm to deal with a crashed two-way decider, have to also work when it doesn’t crash
10. The parallelism argument: no ordering constraint between the two, ergo it runs in parallel, we have two decisions which may not match, violating agreement
11. Ergo we cannot run additional steps if there is a node which is potentially a two-way decider. 
12. But that means we cannot sustain the loss of a two-way decider.
13. Therefore we cannot have two-way deciders
14. Main proof: but if there’s never a two-way decider, well, all one-way decisions have a “remain undecided” case, so it is possible to keep on hitting the remain undecided case forever
15. Therefore we have lost termination. And, termination is why we kept ending up with two-way deciders

-->





<!--

TODO before we tackle FLP, let's go back and try to more formally define the single-leader-replication-without-failover algorithm since it ends up getting referenced several times later.

TODO basically how I want to tackle FLP:

First, give a set of examples we can use to play "find the pattern." Perhaps even reference noticings and wonderings. The examples are:

* Majority voting: 1 red, 1 blue, 5 undecided. All 5 undecided nodes have a "vote red" and a "vote blue" message pending. No one node crash can deadlock the algorithm at this point.
* Majority voting: 1 red, 3 blue, 3 undecided. All three undecided nodes still havea a "vote red" and a "vote blue" message pending. No one crash can deadlock the algorithm here either
* Majority voting: 3 red, 3 blue, 1 undecided. The last node has a "vote red" and a "vote blue" message pending. One crash will deadlock the algorithm, it's now guaranteed.
* Single-leader replication: one leader, one client proposing red, one client proposing blue. If the leader crashes now, the algorithm is guaranteed deadlocked

Then reveal the rule as "you cannot lose a node that is about to decide red vs blue."

Note the fact that the decision is a single step on a single node, and show that the decision is driven by the relative order of messages delivered to that node. We might need to expand on this a bit, I had some prevous ideas here, including

* Diagram: A timeline with "undecided" at origin and "decided" on right hand side
* The same, with a single "deciding step" identified at some point in the middle
* Noting that every step happens on some node
* Noting that 

Anyways, once this is set up we're looking at the relative delivery order of pairs of messages being delivered to each node individually in the algorithm. If we revisit the diagrams, we'll see

* The first diagram (1 red, 1 blue, 5 undecided) results in "undecided" either way, and is not a problem
* The second diagram (1 red, 3 blue, 3 undecided) results in "undecided" or "decided blue" and is not a problem
* The third diagram (3 red, 3 blue, 1 undecided) results in "decided blue" or "decided red" and is indeed problematic
* The fourth diagram (single leader) results in "decided blue" or "decided red" and is also problematic
* Draw conclusion that the rule is: any time the relative order of a pair of messages causes the system to make one of two different decisions, we are le screwed

One tricky thing here is the non-problematic cases contain the problematic case down the line, so you can’t exactly say “not problematic.” But they’re usually able to recover just fine, whereas the problematic cases can never recover. Perhaps you start by listing the problematic case and then show the non-problematic cases are non-problematic except when that specific case occurs. Or maybe you define the bad thing as “deadlocks on this step.” After, if we can fix each individual step that results in deadlock then the resulting algorithm will be deadlock free. Ask the questions “can the algorithm progress?” and “can the algorithm complete?” 

Once we have drawn that rule, we can basically repeat the FLP lemma 3 argument almost directly (with the wording and terminology fixed up) to show what goes wrong in the "decide X vs decide Y" situation. Start with the obvious case, where if you do nothing more and the node is crashed for good then obviously you will never finish. Then show the thing where, if you do more steps and come to a decision, you have a race in your non-faulted case where two parallel threads are going to reach a decision and there’s no reason to assume they will reach the same result, violating agreement.

Now is a good time to link back to “isn’t it weird this happened twice?” Of course it did, this is always what happens!

Then we ask how we keep creating this “decide X vs decide Y” thing by accident, and immediately forgive ourselves when we realize this is a direct result of requiring an algorithm that terminates. If every decision is “X vs undecided” and which case you end up in depends on nondeterministic network message ordering, then there exists a total ordering of network messages in which you happen to always hit the undecided case and don’t terminate. In other words, never having a decide X vs Y step results in an algorithm which is prone to livelock. So we didn’t try to design an algorithm like this because our initial set of requirements precluded this approach entirely.

Conclude we cannot make a fault tolerant consensus algorithm that guarantees termination. 

1. If there is an X-vs-Y decision, the algorithm deadlocks with just one fault
2. If there is no X-vs-Y decision, the algorithm can livelock

Clarify however that we are talking about non-guaranteed termination, not a non-termination guarantee. Pull the FLP quote that suggests a solution out:

<div style="margin-left: 1em; margin-right: 1em; padding-left: 1em; padding-top: .1em; padding-bottom: .1em; border-left: .3em solid #eee; color: #333" markdown="1">
These results do not show that such problems cannot be “solved” in practice; rather, they point up the need for more refined models of distributed computing that better reflect realistic assumptionsabout processor and communication timings, and for less stringent requirements on the solution to such problems. (For example, **termination might be required only with probability 1**.)
</div>

Suggest that we try to find an algorithm where each node local decision is between deciding one thing or staying undecided. Such a thing will not guarantee termination, so it will need to be structured as some kind of infinite retry loop that keeps going until a decision has been made. We just need to make sure the probability of terminating on this round is independent of what happened on previous rounds, so we can show the probability of running N rounds without terminating falls exponentially as a function if N.

Suggest we try to have an infinite number of “voting rounds,” where during each round we pick one proposal to vote on. The result of a round can either be voting in that proposal (thereby terminating the algorithm), or ending the round with the system still undecided. Preview we actually don’t know this route works; but we know the other route definitely doesn’t, so if there’s any hope of inventing a fault tolerant consensus algorithm, this must be the way to do it.

That sets up nicely for the segue where Paxos is borne of a failed impossibility proof, and the idea of eliminating all but one result from the previous rounds

-->









<!--

TODO here's what I had in my phone notes. This is probably all superseded by the new notes above however.

1. We want to terminate, so design the an algorithm that has a finite number of messages and always makes a decision, thereby terminating
2. By induction, one such message must be the “terminator” which has the “decide-if-not-already-decided” semantic we called the one weird property
3. Preview: you cannot have a terminator.
4. Say you design the algorithm such that the terminator is the last step. Then if you lose the receiving node, nobody has made a decision and nobody will. So the algorithm doesn’t terminate after all
5. Say there is more code that runs after and can make a decision. This code must be designed not to wait on the node that receives the terminator; after all, the whole problem is said node can crash
6. But now we have two parallel execution threads in our algorithm that can both make a decision: node A receiving the terminator, and the rest of the nodes running without node A. Both threads decide in parallel, neither can yield to the other, so now two decisions are possible. We have split brain!
7. Unraveling: we can’t have a terminator at the end (not fault tolerant) and we can’t have a terminator not at the end (split brain), ergo we can’t have a terminator. But without a terminator we cannot guarantee termination. So actually a fault tolerant consensus algorithm cannot guarantee termination
8. Conclusion: throw out the termination property 

-->



























<!--





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



-->
