---
layout: post
title: RDMA
author: Dave
draft: true
---

“RDMA” is an umbrella term for a group of networking technologies designed for performance in data centers and supercomputers. I've been working with RDMA networks for a while now, and I've never been able to find a good, straightforward introduction for technical people, so here's one of my own.

## What's in a Name?

The first thing you may want to know about RDMA is that it kind of has a branding problem: the term "RDMA" could mean one of several different (but related) things based on context:

“RDMA” can refer to a programming paradigm involving remote memory access over a network. In short, a 'server' publishes some of its memory to its network peers, which allows those peers to remotely read or write that memory over the network. We'll talk later on about the thinking behind doing this and why it might be useful. In this document, we'll call this definition of RDMA, "RDMA semantics."

But, "RDMA" is also an umbrella term for networking technologies which support RDMA semantics. Which is to say, "RDMA" means "any networking technology that supports RDMA." Not confusing at all, right? Well, you get used to it. In this document, we'll call this definition of RDMA, "RDMA technologies."

There's one more definition that's less common, but you may want to be aware of nonetheless: sometimes people use the term "RDMA" as a way of talking about InfiniBand (a specific RDMA technology) without actually saying the word "InfiniBand." People sometimes shy away from explicitly mentioning InfiniBand in official materials because the name 'InfiniBand' is trademarked. (By the way, please remember to add the implicit &trade; symbol every time you see the word InfiniBand in this document.) Since this is my own personal blog, we'll just use the term "InfiniBand" directly when we want to talk about it.

There's a reason InfiniBand is mixed up in the other two definitions of the term "RDMA" &mdash; InfiniBand was the original RDMA technology (at least, the first prominent technology to support RDMA semantics), and is still a popular RDMA technology today. In the end, just about all RDMA technologies are based heavily on ideas originally outlined in the InfiniBand spec. If you understand InfiniBand, you're 90% of the way to understanding every other RDMA technology out there, even if you never intend to use InfiniBand directly.

And for those reasons, we're going to start this guide by looking at ...

## InfiniBand

To better understand why InfiniBand woks the way it does, it'd help to understand a little about [high-performance computing](https://en.wikipedia.org/wiki/Supercomputer) (HPC) community and what it does first.

HPC is all about building supercomputers &mdash; really big computers that can do a lot of computation very quickly. For the most part, supercomputers are built using the same basic principles and components as regular computers: to make a supercomputer, you employ "[massive parallelism](https://en.wikipedia.org/wiki/Massively_parallel)," which you can think of as a fancy way of saying you build big computers by linking lots of smaller computers together and having them work together. The smaller computers are often called 'nodes,' and the network of these nodes linked together is often called a 'cluster.' Although this might seem simple in principle, in this case the devil is in the details.

One of those details is network overhead. Whenever a program running on a node needs to exchange network information with another node, it needs to run some code that sets up and manages the transfer. This network code runs on the same CPU hardware as the 'real' computational code the supercomputer was built to run, so any time the network code is running, it's taking up time that could have instead spent doing those computations. Even though this overhead is usually pretty low, it can become a barrier to scaling the cluster; after all, if you're adding $O(N)$ times as many nodes to an existing cluster, you're protentially creating $O(N^2)$ as many network transfers!

Minimizing network overhead can thus become critically important to scaling a supercomputing cluster past a certain point &mdash; which is where technologies like InfiniBand come in.

### History

[InfiniBand](https://en.wikipedia.org/wiki/InfiniBand) was introduced around the turn of the millennium in order to solve exactly the kinds of scalability problems we discussed above.

At the time, large-scale computers were starting to near I/O bandwidth limits, not only in the networks that link nodes together, but also in the [hardware interconnects](https://en.wikipedia.org/wiki/Bus_(computing)) that link together the different components of a computer. InfiniBand was introduced as one of several competing standards attempting to address these bottlenecks (and was, itself, a merger of other competing standards).

The InfiniBand folks chose to invent a completely new technology to replace the dominant networking and interconnect standards at the time: [Ethernet](https://en.wikipedia.org/wiki/Ethernet) and [PCI](https://en.wikipedia.org/wiki/Peripheral_Component_Interconnect), respectively. Although InfiniBand was in no way interoperable with these technologies, it did provide a handful of benefits, primarily much higher bandwidths than the competing technologies of the time could provide, with much less software overhead involved in networking operations.

InfiniBand came with a lot of lofty goals and created a lot of hype at the time. Many of the ideas it cooked up never really caught on, but the networking model did catch on in a big way in the HPC community. As we discussed earlier, the people building supercomputers are more focused on reducing network overhead and scaling to higher bandwidths than most computer users. InfiniBand's lack of interoperability with legacy technology was not a oncern to HPC people, who are generally starting with an empty warehouse and filling it with all-new computing hardware anyways.

### How It's Used

We mentioned above that InfiniBand was envisioned as a replacement both for networks (which connect nodes together) and hardware interconnects (which connect the components of a single node), with the goal of blurring the line between the two  to make it easier to build computers with exotic hardware architectures. This thinking influenced how InfiniBand was designed, and (as we'll see later) is arguably where the idea for 'RDMA semantics' came from in the first place.

That all said, in this document we're going to consider InfiniBand purely as a networking technology &mdash; which, to the author's knowledge, is how a vast majority of InfiniBand's users treat it anyways. If you choose to further research InfiniBand on your own, however, you may find it useful to understand that the designers were trying to build something beyond a networking technology.

### How It Works

The main design goal behind InfiniBand is to **minimize network processing in software code**. This would mean CPU time that previously had to be used for networking can now be used to run more computation, which is what we built the computer to do in the first place! Less obviously, removing network code potentially allows the network to run faster: there are fundamental limits to how fast CPUs can start and run networking code, so cutting the CPU out of the loop removes this potential bottleneck.

How are we going to remove network code? We have two main tools: we can either push the software logic 'down' to the hardware level, so it runs directly on the network card, or we can reconfigure the network such that no processing logic is needed at all. InfiniBand does a little of both.

Here are the basic design ideas. We'll look at each of these in greater detail throughout the article, so don't worry if the overview doesn't make sense out of context. Let's start at the bottommost layer and move up:

**Custom physical specs and signaling protocols**: The details aren't important to our discussion; just keep in mind that InfiniBand cables and ports don't work with anything that came before InfiniBand, and they support higher bandwidths than did many contemporary technologies.

**Switched network fabric**: The only valid way to connect nodes in an InfiniBand network is using a [network switch](https://en.wikipedia.org/wiki/Network_switch); other schemes like shared media and repeater hubs ([discussed here](https://en.wikipedia.org/wiki/Ethernet#Evolution)) are not supported. This lets InfiniBand assume each cable connects exactly one transmitter to exactly one receiver in each direction, with no need for [collision avoidance](https://en.wikipedia.org/wiki/Channel_access_method). Every connection between two InfiniBand peers passes through a single path through the network.

**Lossless networking**: InfiniBand peers and the network switches that connect them proactively cooperate to make sure nobody (peer or switch) gets overloaded, which means nobody ever needs to drop packets due to network congestion. That the network that never drops packets in any expected case why InfiniBand networks are often called 'lossless.'

**Segmentation/reassembly in hardware**: All networking technologies split messages into [smaller units](https://en.wikipedia.org/wiki/Protocol_data_unit) that are each transmitted along the network; Ethernet calls these units 'frames,' and IP calls these units 'packets.' On conventional networks, applications typically rely on a software implementation of TCP to segment buffers for transmission and reassemble transmissions on the receiving end; InfiniBand instead does this in hardware.

**Queue-based interface**: Ultimately, software initiates all network data transmissions, so it doesn't matter how fast your network is if the software interface to use the network is slow. Instead of something like BSD sockets, InfiniBand uses an interface based on queues to provide a low-overhead interface for software to use the network, with support for aggressive batching to further reduce overhead. On many implementations, this hardware interface can be accessed directly by userspace applications via memory reads/writes, bypassing the operating system kernel entirely to further reduce software overhead for network operations.

How do we use all of these things from software code? RDMA networking hardware provides two basic modes of operation, which can be mixed and matched as needed:

**Send/Receive semantics**: A simple message-based scheme that might look familiar to anyone who has programmed conventional networks before. The **send** operation accepts a message buffer to send, and completes once the buffer has been sent on the wire. The **receive** operation accepts a message buffer to receive into, and completes once something has been received into the buffer. You can think of this sort like UDP, but with support for very large datagrams (megabytes or even gigabytes!).

**RDMA semantics**: A scheme which grants remote read/write access to part of a node's local memory. A 'server' node uses the **register** operation to make a slice of its memory available for remote access. Clients can then use **read** and **write** operations to read/write the registered server memory remotely, over the network. The server's network card performs all read/writes by itself, without sending a notification to software, which allows the server's CPU to do other work by itself.

Keep in mind that these aren't high-level APIs; these are the lowest-level operations that the hardware provides to software. Providing software with such a high-level interface minimizes the amount of processing that has to be done in software to translate a high-level API call to something the network card can start processing.

If any of this went too fast, don't worry &mdash; we're going to circle back to each of these ideas in a bit more detail below. Let's start at the lowest layer, with how InfiniBand networks are built out of network switches.

## Switched Fabrics

> This is a case where InfiniBand has *less* functionality than its contemporary peers. So it's worth looking at what conventional networks support that InfiniBand doesn't, so we can see why to deliberately drop support.
>
> Earliest networking technology envisioned as 'shared media' &mdash; basically, connect a bunch of cables together with splitters. Lots of machines on the network.
>
> The problem you need to deal with here: collisions. Ethernet is just a bunch of impulses &mdash; an impulse is a binary 1 and a lack of impluse is a binary 0 &mdash; which means you can only have one transmitter on the network at a given time, and all transmissions go to all peers, who then have to figure out whether or not the transmission is for them.
>
> What if you have multiple transmitters trying to use the network at the same time? Then you have a collision. Many strategies for detecting and resolving collisions so that everyone can use the media fairly. Link to some.
>
> Okay, so nobody builds Ethernet networks out of shared cables and splitters anymore. But we do still (sometimes) use hubs, which are just repeaters. Whenever a signal is received on one wire's incoming line, the hub transmits that signal on all outgoing line. All the collision stuff applies there too.
>
> And, fun fact: this also applies to wireless networks, because Wi-Fi is just Ethernet, except with radio impulses over the air instead of electrical impulses over a cable. All the same collision stuff totally applies.
>
> Anyways, the world as we've described it so far is kind of inefficient.
>
> One problem is collision avoidance: if you make it *proactive*, then unused time slices are wasted. If you make it *reactive*, then you waste some of your time detecting collisions and stopping transmission. Eventually, as you grow the network to a certain size, you end up with so many peers that in theory, the collisions should dominate over, you know, actually useful traffic.
>
> We're also wasting a lot of bandwidth too. Say you have a hub with four machines, A, B, C and D. Say A wants to send a frame to B and C wants to send a frame to D. Ideally we should be able to send both of those frames in parallel; but instead, our network only supports one frame at a time; so either A has to wait for C or vice versa. And, a lot of frames are being repeated to nodes who are ultimately going to ignore them.
>
> So, for high-performance networking applications, we figured out a better way to do things a long time ago: network switches. A switch is basically a smarter hub: instead of repeating an incoming frames to *all* outgoing lines, it figures out who the frame is addressed to and which port that peer is connected on, and transmits the frame *only on that port*.
>
> Bandwidth wastage? Gone! The frame only goes on the port to which the recipient is connected.
>
> Parallelizable? Yup! As long as two frames are addressed to different transmit ports, the switch can totally transmit them in parallel.
>
> Now imagine for a second that the *only* way you connect peers in the network is using switches; if you have more peers than you have ports on a switch, then you join switches together with more switches, forming a hierarchy. Now you get something interesting: a network that has no Ethernet collisions! Because each connection in the network is either between one peer and one switch, or between two switches. 
>
> Mention Clos networks as a practical way of realizing this.
>
> Back to InfiniBand: InfiniBand deliberately *does not* support any kind of shared media and does not support collisions, so the only way to build an InfiniBand network is with switches. There simply isn't such a thing as an InfiniBand hub.
>
> Why? Well, (a) switches were already cheap and ubiquitous by the time InfiniBand came around, and (b) it obviates the need for a bunch of complicated, performance-sapping collision logic.

## Lossless Networking

> To understand 'lossless' networking, we have to understand what makes networks 'lossy'
>
> Let's say you have a switch with four ports. That means you have 4 receiving lines and 4 transmit lines. Assume, like pretty much all switches out there, the lines are balanced: all top out at the same maximum bandwidth. Now let's say ports A and B are using 100% of their bandwidth to transmit to line C. Now what?
>
> Look at this for a little while and it'll quickly become clear that the *network* physically can't fix this problem. You might think buffering is the answer, but (a) you can only buffer so long before this problem comes back, and (b) buffers drive up latency (link out to bufferbloat). So, once we're in this situation, the switch really has one option: transmit as much as it can, and forget about anything it couldn't.
>
> This situation is called 'congestion' (link out). 
>
> One way or another, A and B will have to slow down enough that sum(incoming on A + incoming on B) is leq (outgoing on C). The obvious policy is for A and B to each ratchet down to 50% link utilization, but in some cases you might want to prioritize one over the other (link out to QoS).
>
> So how do A and B avoid overloading switch? On a conventional network, this is typically done *reactively* by the peers. Peers send as much data as they feel like, and watch for signs that their transmissions might not have reached the remote peer. If they detect loss (or what they think is loss), then they slow down until they stop losing packets, then slowly start to ramp up again until they hit packet loss again.
>
> Link out to two generals, which arises directly out of this decision.
>
> One thing that's nice about this scheme is that it's robust &mdash; A and B never have to communicate with one another, they can be completely unaware of one another, and they don't even need to have the same backoff and ramp up policies. That's why this kind of reactive adjustment is how computers utilize conventional networks, and the Internet with it.
>
> InfiniBand instead takes a *proactive* approach, meaning A, B, C and the switch all work together to make sure that the switch never gets overloaded in the first place.
>
> Here's one possible scheme: bandwidth reservation. 
>
> A slightly more robust scheme: credit schemes
>
> Another idea: explicit congestion notifications and backpressure
>
> In theory, other things are possible. AFAIK, modern deployments typically use credit schemes between the peers and ECN at the switch, but don't quote me. (Also research this before making any statements.)

## Hardware Frame Assembly

> Explain what TCP does in the conventional stack, in light of 'lossy' networking tech like Ethernet
>
> Here's the problem for the IB designers: TCP is necessary because it's the semantics applications want, but the network doesn't support; and it revolves around complex rules with a lot of branching, which makes it challenging to offload in software (at least with the tools that were available at the time).
>
> So, we're going to have to make TCP 'go away.' The network has to support the semantics that applications want out of TCP.
>
> The first part is already done for us: the network is lossless, so you can't lose frames and they never need to be retransmitted.
>
> The second part is preventing reordering. Here's the idea: when we're setting up an IB conection, we will negotite a single path through the network between the peers. (If we're using a reservation-based lossless networking scheme, we already had to do this anyways!) If we can guarantee all frames flow through the same path, then the transmit order will be the same as the receive order, because there's no way for frames to 'race'
>
> So then how does the hardware segment and reassemble frames? Now it's pretty easy, because the only thing we need to do is make sure frames haven't been lost or reordered. Here's an example scheme:
>
> Both peers know of a connection identifier. Each tracks a sequence number for the connection. Each frame is transmitted with the transmitter's current sequence number, which is incremented upon sending the frame. The receiver makes sure the received sequence number matches the local sequence number before incrementing its own.
>
> What happens if a frame is received out of order? Well, in InfiniBand that shouldn't happen, so some hardware must have faulted. We respond by terminating the entire connection. Applications never see drops or reorders: they either see a reliable network or a disconnect, nothing in between.

## Queue Pairs

> Alright, so I'm a little shaky on the specifics, and I'm not sure exactly what we want to include here. 
>
> * A connection between two peers is a **channel**
> * Each endpoint consists of a hardware-allocated **queue pair** (QP)
> * A QP is a **send queue** (SQ) and **receive queue** (RQ)
> * Each queue is a circular array of **work requests** (WRs)
> * You place a WR on the SQ to send something
> * You get a WR on the RQ when you receive something
>
> This interface is designed to provide *ordering*, *batching* and *kernel bypass*.
>
> Ordering: remember from earlier that our network is lossless, and that all frames on a channel take the same path through the network, so it's strongly ordered too. The order you put WRs on your SQ is the order the NIC will transmit your data. So, ultiamtely, you control the order in which work will be carried out. As we'll see later, applications sometimes need to take advantage of those ordering guarantees when utilizing the network.
>
> Batching: Each trip between the CPU (where the software is running) and the network card (which is the thing actually connected to the network) is expensive. It's particularly bad when the CPU needs to get a notification from the networking card. Explaining what interrupts are and give some ballpark numbers about the cost of just *one* interrupt. The idea of a queue interface is the ability to batch several commands to the network card in one ago, and to be able to receive a batch of incoming stuff 
>
> Kernel bypass: This system is designed so that vendors can implement the interface *in hardware*. Allocate QP in network card memory and publish it to the CPU's address space via memory mapping. The kernel can then use virtual memory addressing to map the queue memory into the process's address space. Now the userspace process can directly interact with the QP without any help from the kernel. That's good, because calling into the kernel from userspace requires a software interrupt, and we already know how slow those are!

## Verbs

> Details on the six main things you can do: send/receive, register/invalidate, read/write
>
> If we want to explain any basic examples that illustrate patterns, that might be good. At least we should hint that send/receive is the only way to tell other nodes what memory addresses you've shared with them.

## Beyond InfiniBand

> InfiniBand was just the original spec. Its features &mdash; a switched fabric, lossless networks, hardware frame segmentation, a queue-based API, remote memory read/write semantics &mdash; are all individual features that you can mix and match with more conventional technology.
>
> Converged Ethernet and Data Center Bridging extend Ethernet to allow lossless, switched Ethernet networks using ECN and backpressure.
>
> RoCE as a hybrid which encapsulates InfiniBand frames on top of CE. In theory, lossy Ethernet traffic can then coexist with lossless. One challenge here is ECN only applies to RoCE traffic, so bursts of lossy traffic can trash your lossless traffic unless you do something special. (And I don't really know what special is in this case.)
>
> iWARP which employs RDMA semantics on top of a hardware implementation of TCP. (IIRC.) Put this on top of CE and you get something a lot like RoCE without having to take up a bunch of InfiniBand.
>
> There's probably more stuff out there

## Why "RDMA?"

> Resurrect old content from git explaining DMA lightning-fast and how it gives rise to 'Remote DMA,' especially if you want the network to be an interconnect like PCI and vice versa.
>
> Note that we already covered interrupts, memory mapping and virtual addressing back when we discussed QPs, so we may need to cover a little less here

## Strengths and Weaknesses

> The main goal is performance.
>
> Minimal interrupts
>
> Zero copy
>
> It tends to be *easier* to program. If you've ever tried to implement message framing over a TCP connection, you'll probably appreciate how RDMA does this for you
>
> Some other basic tenets suffer, making it challenging to deploy RDMA in a variety of scenarios.
>
> Security is tricky, especially with RDMA semantics, where the memory is now exposed to the network. IB *does* include protection mechanisms, but recently we've seen attacks on CPUs and virtual memory, and it's likely you can carry out the same sort of attacks on RDMA cards. Security holes cannot be easily patched or mitigated because so much has been pushed down to the hardware level. No TLS / encryption support, you'd have to do this at the application level yourself.
>
> None of that is a concern for a private cluster of private machines running your own home-grown code though. It's only a concern if you try to hook up an RDMA deployment to a wider, less trusted environment.
>
> Resiliency can also be a concern. In-hardware frame segmentation/reassembly assumes the network is lossless because we proactively avoided congestion, but congestion isn't the only source of loss in a real network. You can also lose packets due to bugs in network switches and physical problems with hardware (even with something as simple as a mismanufactured cable). A reactive stack will treat loss from faults the same as losses from congestion, but RDMA technologies will generally abort connections at the first sign of a fault. Intermittent / ongoing faults can cause connections to be torn down and recovered repeatedly.
>
> For smaller deployments this may not be a concern, but if you're going big, plan to have some sort of quality control on your hardware, and/or infrastructure for identifying these kinds of problems in the wild
>
> Scalability: TODO Sam's up

## Conclusion

> Let's start building a pile of 'further reading' links. For now, these are links *I'm* using; we'll pick out the ones worth keeping once the article is done
>
> https://www.mellanox.com/pdf/whitepapers/Intro_to_IB_for_End_Users.pdf
>
> https://www.afs.enea.it/asantoro/V1r1_2_1.Release_12062007.pdf
>
> https://www.ieee802.org/1/files/public/docs2014/new-dcb-crupnicoff-ibcreditstutorial-0314.pdf

