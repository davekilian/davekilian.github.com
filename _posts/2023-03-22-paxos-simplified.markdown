---
layout: post
title: Paxos, Abridged to the Point of Usefulness
author: Dave
draft: true
---

This post is an introduction, or reintroduction to the distributed consensus algorithm *Paxos*. If that name is familiar, you probably know to utter it only with a certain reverent tone, one that implies you hold an appropriate fear of the algorithm. Paxos is infamous for being incredibly difficult to understand; in fact, it's modern counterpart, [Raft](https://en.wikipedia.org/wiki/Raft_(algorithm)), was invented in a [search for a more understandable Paxos](https://raft.github.io/raft.pdf).

## Why is Paxos so hard?

I have a pet theory: the funny archaeology metaphor original [The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) (TPTP) paper which first introduced Paxos to the world was only ever clear to contemporary researchers who ...

* Work in the space of fault-tolerant distributed algorithms
* Are already deeply familiar with two related problems: *Consensus* and *Atomic Commit*
* Grok an atomic commit algorithm called *Two-Phase Commit* or *2PC*
* Have already tried, and failed to find a more fault-tolerant alternative to 2PC
* Know of the [FLP result](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf), what it proves impossible, and what else it casts doubt upon
* Are starting to give up hope *any* fault-tolerant atomic-commit or consensus algorithm exists

Someone already that deep in this space would immediately understand the framing metaphor used in TPTP, and would be delighted to find that practical consensus algorithms not only exist, but are actually simple extensions to what we already had. Paxos is basically a variant of the two-phase commit algorithm which replaces the central coordinator (a single point of failure) with a peer-to-peer majority-rules voting scheme. The framing metaphor used in the paper, then, is a sort of magician's flourish, as if to say, "See? Not only is consensus solveable, but *you knew how to do it all along*!"

Unfortunately, you and I are not said contemporary researchers, so in trying to read the paper, we're left with the dual problem of understanding how the algorithm works within the archaeology metaphor *and* how that metaphor maps to computers and networks. For most of us just getting started in the space, this is simply too much. 

In this post, I'm going to introduce, or maybe re-introduce you to Paxos by fast-forwarding through the historical context into which Paxos was first introduced. Then I'm going to explain Paxos, without the metaphor, using the context we just built up. My hope is for you to come away with a pretty good sense of the core ideas that make consensus algorithms tick, and maybe then have an easier time reading key papers like TPTP and FLP if you so wish. Enjoy!

## The Atomic Commit Problem

TODO start with atomic commit because it better maps to what normal people think about when they think distributed systems. Then show how you solve it on a single machine: log, then apply, with the atomic commit point being the point you applied. Then talk fault tolerance, if you crash during this algorithm, then the atomic commit point changes to whether or not the change made it to disk durably. 

Then we need to change gears and talk about distribution. Show some naive things that don't work, then talk about the two generals metaphor, and come back to gee, it doesn't really seem like this is easy. For now, let's leave this problem open, just as it was back in the day.

## Consensus: A Related Problem

TODO we need to lay out this problem concisely and precisely while also relating it to the work we did in the context of atomic-commit. I'm not positive exactly how to link the two, however. If there's some kind of duality / reduction mechanism that exists, we should talk about that.

Also, this is the point at which we should talk about desugaring the problem, from "replicating a data structure in the face of faults and conflicting updates" to "replicating a single variable" and repeating that to form a log of updates.

## Two-Phase Commit

TODO introduce as prior art for consensus research and explain it as a viable solution to the distributed atomic commit problem. Explain the basic mechanics and show how it admits faults with individual nodes. However, then show the mess you get when you get a coordinator fault. How can we create a consensus, or equivalently, an atomic-commit algorithm that tolerates a fault in any node? How can we eliminate the single point of failure?

## The FLP Result

TODO well, we actually know we *can't* do any better in certain situations. FLP showed there does not exist a fault tolerant consensus algorithm in *asynchronous networks*, which are ones where there are no *failure detectors*, which in practice basically means "there are no timeouts." If you require the algorithm to wait for a transmission on an infinite time scale, if there is truly no way to tell whether or not a transmission was lost, then there's no way to force a fault-tolerant algorithm to ever make progress.

This is an airtight impossibility proof for a kind of network that doesn't exist. But it's scary. What if the same line of thinking can be extended to networks that *do* have failure-detectors? That is, can we extend this proof to figure out 

Per his blog (link), that's exactly what Lamport was *trying* to do; but then it turned out that dead-end in proving consensus is *impossible* was actually a lead on a *proof by construction* that such algorithms do exist. The resulting algorithm is Paxos. The underlying realization is that we've had distributed, asynchronous atomic commit algorithms in the real world for hundreds of years. The answer has been sitting right there under our noses!

## Atomic Commit in Real Life

TODO so here's an atomic commit protocol that you've almost certain participated in (or, you've had the opportunity to participate in but didn't, tsk tsk). When we want to make a democratic election, we vote. In the United States, most votes are first-past-the-post majority rules. It's asynchronous in that everyone cast their votes alone, independently, with no ordering constraints relative to other people. It's distributed in that everyone votes. And it's an atomic commit: at the end, there is one victor.

How exactly *is* voting an atomic commit protocol? Every atomic commit protocol needs to have some kind of commit point, a single action, taken by one person, that moves the system from the "old" state (undecided, in this case) to the "new" state (a victor exists) without the possibility of ever going back to the old state, even temporarily, or having intermediate states which are neither the old or new state.

In a majority-rules election, that atomic commit point is essentially whoever's vote pushes the winner's vote count over the 'majority' threshold. That is, if you let $N$ be the number of people who will cast a vote in the election, then the winner is whoever gets at least $N/2 + 1$ votes, and the atomic commit point is the vote that pushes a candidate across that threshold.

The curious thing is, because everyone votes independently, the person who casts this critical vote has no idea their vote is the critical vote! We can identify it by taking a global view of the entire system, including all people and all votes; but on the ground, nobody has this global view of the system. We'll never know whose vote was the atomic-commit vote, but we can still find out asynchronously, by tallying the votes, which candidate is the one we're all committing to.

This mechanic is key to the way Paxos works. We're going to "vote" using a two-phase commit protocol, but there will not be a central coordinator. Instead, each node will independently cast its own vote, and independently tally the results of all other nodes' votes. Once a candidate has received $N/2 + 1$ votes, the operation is committed, allowing us to move into the second phase of the commit. 

## Paxos: Majority-Rules Two-Phase Commit

TODO now we lay out the basis of the algorithm. Before going deep, explain first that the *only* thing we're taking from the metaphor above is the idea of casting votes independently and then asynchronously finding out who won by tallying votes. Everything else is closer to 2PC than it is to a real-world election.

TODO define safety and liveness properties. The locked voting protocol provides safety, once a majority of locked-in votes are for the same candidate value, we can't renege. The nodes that have locked in their votes will never unlock their votes unless a different candidate reaches a majority, and a second majority can never form, no matter who fails and what happens. It provides fault-tolerant liveness as well: as long as $N/2 + 1$ nodes are online, the network can still in principle pass decrees.

TODO An important micro-optimization is a gossip protocol by which nodes inform each other the election ended and name the victor; nodes receiving a victory notification take this as gospel instead of independently tallying the votes. This works as long as you control the code and you ensure this message is only sent if a node sees a majority. But some people find this offensive, leading them down the road of Byzantine fault tolerant algorithms, which leads to Mickens's saddest moment.

## Paxos: Better with Leaders

TODO we're not quite done however. What we described above is the core peer-to-peer Paxos algorithm. It has a problem in practice: split votes.

Imagine three different decrees all start at the same time. They all race to become the winner, but in the process, each gets about a third of the votes. Now we're stuck: nobody has a majority, and everyone's vote is locked, so there's no way for any one candidate to get to a majority. We failed to come to consensus. This failure is detectable (phew!) but it's permanent nonetheless. We can't have nodes change their votes because that was key to ensuring safety properties earlier.

A scheme to fix this is to ensure every candidate decree runs unopposed. To do that, you need one node to receive all candidate decrees, detect conflicting updates, and resolve them by choosing one to keep, and throwing away the rest. The chosen update is then proposed to the rest of the network, and the rest of the algorithm executes as described in the previous section. This is still the same Paxos algorithm, but now we've identified a special "leader" node that resolves all conflicting updates ahead-of-time, so that every decree runs unopposed, such that you can never have a split vote.

But wait, this leader is a single point of failure! Our new leader-based Paxos has a bootstrapping problem. You need to figure out who is the leader, and if they go offline, you need to figure out who will be the leader next. How do we come to consensus on who will be the leader? Why, of course, with Paxos! We can use Paxos to have nodes elect their leaders.

But it can't be *that* simple. Consider a naive approach where every node, upon realizing the leader has faulted, proposes itself as the leader. Well, obviously, if every nody locks in its own vote as itself as the leader, then you get a split vote. In practice, you need some kind of external backoff heuristic; for example, you might make a list of nodes, including yourself and your peers, and sort by IP address. The first node that appears to still be online is the one you propose to be the leader.

This is the point where Paxos really starts to spiral out of control in terms of complexity. The core Paxos algorithm was really only ever meant to handle peer-to-peer voting. Juggling leadership elections while leaders are putting out conflicting decrees gets tricky, fast. 

## Raft

