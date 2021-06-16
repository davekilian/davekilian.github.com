---
layout: post
title: RDMA (deprecated)
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

## A 'Switched Fabric'

Most data center and supercomputer neworks are wired in a pattern often called a 'switched fabric.' This means something much less fancy than it sounds: it just means all the computers are wired together using network switches.

A [switch](https://en.wikipedia.org/wiki/Network_switch) is a wired networking device. The front face of a switch consists of several network ports, each of which is connected to something else (a computer or another switch, usually). In a process called [packet switching](https://en.wikipedia.org/wiki/Packet_switching), a switch listens for incoming network transmissions on every port, examining each to determine what network address each incoming transmission is intended for; the switch then determines which port is connected to said address, and retransmits the data to that port.

The de facto standard way to wire up a large network like for a supercomputer or a data center is to hook up your computers together using a network of switches. In an ideal world, you can hook up all the computers to a single switch, but if you have enough computers to fill a warehouse, you'd have a hard time finding a switch with enough network ports to accommodate them all; instead, you can build a 'virtual' switch of the required size using smaller switches. (If you're interested in how to do this, see [Clos networks](https://en.wikipedia.org/wiki/Clos_network).)

This network of switches used to interconnect the nodes of your supercomputer / data center / what have you is often called a *fabric*. The term is a bit of a buzzword, so it's hard to pin down a crsip definition, but you can loosely think of a fabric as a network that facilitates high-speed communication between many computers. The term originally came from the diagrams of data center style networks, which often contain regularly repeating criss-crossing elements that remind some people of the interwoven threads of a piece of fabric.

If you crack open the InfiniBand spec (maybe some light bedtime reading?), you'll find that the term 'switched I/O fabric' appears right at the beginning of the spec. InfiniBand assumes you will build your network as a network of switches that connect a bunch of computers together. Interestingly, this wasn't a new idea what InfiniBand came around &mdash; this had already been the standard long before InfiniBand came around &mdash; but InfiniBand went a step further in *only* allowing networks to be build this way.

In the next section, we'll see how this allows InfiniBand to push additional functionality on the switches in order to build a new kind of network ...

## Lossless Networking

RDMA technologies generally use a technique called 'lossless networking' (InfiniBand included). To understand what it means for a network to be 'lossless,' we first need to understand what it means for a conventional network to be 'lossy.'

### Network Congestion

Congestion is a situation which occurs when transmissions are pushed into any link faster than that link can transmit. The result is a situation where at least some of those transmissions cannot be delievered.

Let's illustrate with an example:

Most networking technologies use [full-duplex](https://en.wikipedia.org/wiki/Duplex_(telecommunications)) cabling with separate transmit/receive lines at each end. This is just a fancy way of saying the cable can carry transmissions in two directions, and usually accomplished by having two separate internal cables wrapped up inside the plastic sheath. Usually, the lines are balanced, which means they both transmit at the same rate. Let's say you have a 1 gigabit network device / cable: that means something plugged into the cable can simultaneously transmit and receive, each at a rate of 1 gigabit (1 billion bytes) every second.

Let's say you have a switch with four full-duplex gigabit ports, which we'll label $A$, $B$, $C$ and $D$. What happens if both $A$ and $B$ transmit at a rate of 1 gigabit per second, and all of those transmissions are directed at port $C$?

Clearly we have a problem: the switch's job is to forward each of those incoming transmissions from $A$ and $B$ and transmit them all to $C$, but the switch is receiving 2 gigabits of data per second, and can only copy those forward at a rate of 1 gigabit per second. Port $C$ is overloaded (by a factor of 2, in this case), and the result is what we call 'congestion.'

What are we going to do about this?

One trick that's been tried before is buffering: we transmit as much as we can, and the store the remainder in some buffer memory so we can transmit it later. This isn't a silver bullet: if $A$ and $B$ sustain their high transmit rates long enough, eventually the switch will run out of buffer memory, and we'll be left back where we started. Even if we don't run out of memory, buffering still turns out not to be great: large network buffers are [known to degrade performance](https://en.wikipedia.org/wiki/Bufferbloat).

If not buffering, than what? Perhaps unsatisfyingly, the short answer is: nothing. The best answer we've come up with over the decades is [best-effort delivery](https://en.wikipedia.org/wiki/Best-effort_delivery): forward along as many of the transmissions as port $C$ can handle, and forget about the rest of the transmissions. Once a switch is in this situation, there isn't anything better you can do.

So the question becomes, how do we avoid getting into this situation?

### Congestion Control

One way or another, $A$ and $B$ will have to slow down so that the sum of their transmit rates is less than $C$'s maximum transmit rate, which is 1 gigabit per second. That *probably* means we want $A$ and $B$ to both transmit at half a gigabit per second, but there are some applications where you might want one to take priority over another. (We won't worry about that here.)

Conventionally, this is done 'reactively' in software. As it sends data over the network, the software watches for signs that some of its transmissions may not have reached the remote peer. Any time a transmission appears to be lost, the software assumes some link must be congested, so it ratchets down its transmit rate and retransmits the data that was lost. Over time, the software slowly increases its transmission rate in the hopes that the network has become less congested, until it once again sees transmission losses and has to ratchet back down. In this way, all users of the network are always 'feeling around' for a transmission rate that the network can handle right now.

There are some nice facets to this scheme. The first is that it does indeed work &mdash; $A$ and $B$ will eventually negotiate their way to a net transmit rate that $C$ can handle. This scheme is also robust: $A$ and $B$ don't have to explicitly communicate with each other, or the overloaded switch, to figure out what their transmit rates need to be. In fact, they don't need to share anything at all: they can all use completely different implementations with unrelated backoff and ramp-up policies, and things will generally work.

The downside is that the software that manages transmission rates and monitors transmissions is complex and heavyweight: you need a full software implementation of [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) and all its complexities running in between your application code and the network hardware. If you're building a supercomputer, you'd want to look for an alternate approach with less software!

### Proactive Approaches

RDMA technologies generally pick more 'proactive' approaches to talking this problem. In other words, $A$, $B$, $C$ and the switch will all work together to make sure the link never gets overloaded in the first place.

Here are some basic approaches you might take to implementing this:

**Bandwidth reservation**: Have the switch allocate some of $C$'s outgoing bandwidth when, e.g., $A$ establishes its connection with $C$. The switch notifies $A$ how much bandwidth it actually gets, and $A$ makes sure never to transmit faster than that. The switch reserves that portion of $C$'s bandwidth exclusively for $A$'s connection to $C$, so as long as $A$ stays below that transmission rate, it can be certain there will always be available bandwidth. Simple, but inflexible!

**Credit schemes**: Improve on the above by dynamically reserving bandwidth using a system of 'credits.' Each peer/switch in the network grants credits to its neighbors; each credit permits that neighbor to transmit a certain number of bytes back to the original peer/switch. Every transmission 'uses up' some of those credits, and if a peer/switch runs out of credits for a given link, then transmissions along that link must wait until more credits have been granted. This way, if a device starts to get overloaded, it can throttle traffic before actually reaching its limits, by simply reducing the rate at which it grants credits to its neighbors. ([The devil is in the details](https://www.nap.edu/read/5769/chapter/4#9), of course.)

**Congestion notifications**: A different approach entirely! Peers and switches operate on a best-effort basis, but when a link starts to get overloaded, the device sends back a 'pause' command on that link, directing that neighbor to stop transmitting. Once buffers have been cleared, the device can then send a 'resume' command to unblock the link and resume receiving transmissions. 

In practice, these are all elements of a complete solution; credit schemes and congestion notifications can coexist in a single implementation, for example. This list is also by no means exhaustive.

Note that these schemes all require peers and switches to maintain state about each other, and every peer must abide by the limits set by their neighbors &mdash; even one peer transmitting more than its allotted share can throw off the whole system. You would never put something like this on the Internet, because you can't trust everyone on such a large network to cooperate with you so nicely; but these schemes work fine in a data center, where you own and operate all the equipment yourself.

### In InfiniBand

As we previously alluded, InfiniBand uses a lossless networking scheme like the ones we just described. (The specification outlines a credit-based scheme but leaves many of the implementation details to the networking hardware vendor.) This decision lets higher levels of the stack make a few key assumptions:

First, becuase InfiniBand networks are 'lossless' in that links don't ever reach the point where transmissions have to be dropped due to congestion, peers can assume that all transmissions will be delivered. A lost transmission is a sign of a more serious error, and is typically grounds for terminating the whole connection.

Second, in order to set up these lossless connections, an InfiniBand fabric first needs to negotiate a path through the network between the two peers, in order to set up things like flow control credits. This means, in the end, there's only *one* path through the network for all transmissions along that connection. In the absence of any way for transmissions to take different paths and 'race' with each other, this allows InfiniBand to guarantee that all transmissions arrive at the destination in the same order they were sent.

So, in short, Infiniband guarantees reliable, in-order delivery of data transmissions. This has important ramifications for the next layer of the stack ...

## Hardware Frame Assembly

> Explain what TCP does in the conventional stack, in light of 'lossy' networking tech like Ethernet
>
> Here's the problem for the IB designers: TCP is necessary because it's the semantics applications want, but the network doesn't support; and it revolves around complex rules with a lot of branching, which makes it challenging to offload in software (at least with the tools that were available at the time).
>
> So, we're going to have to make TCP 'go away.' The network has to support the semantics that applications want out of TCP.
>
> But actually, we did this already! The network is already lossless, so you can't lose frames and they never need to be retransmitted. And, in order to set up a lossless connection, we already had to forge a *single* path through the fabric between the peers. So all frames flow through the same path, losslessly, which means there isn't a way frames could arrive out of order &mdash; they just can't 'race.'
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

## InfiniBand &mdash; Recap

A slightly different way of doing this section that might be enlightening:

The über-goal is to minimize software overhead. Give CPU cycles back to the software, and allow the CPU and network to scale independently

What is the network code doing, that we want to remove?

It handles transmission reordering, loss and rate limiting due to congestion. We'll build a self-rate-limiting network that never reorders or loses transmissions, so that this problem 'goes away' from the software's perspective.

It handles segmenting large messages into smaller transmissions and reassembling them. We'll do this in hardware.

It virtualizes the network card, so multiple programs can share the NIC without affecting each other. We'll create a virtual interface in hardware and expose it to userspace programs using mapped memory buffers.

TODO probably want to mention RDMA semantics here too ...

Do all that and you have InfiniBand.

I wonder also whether we want to rejigger the intro section on 'how InfiniBand works' to cover more or less that same stuff, but at a higher level with no jargon

---

So, what did we learn?

> Recap the basic design points

What was the point of doing all this?

The software overhead for using the network has now been reduced to a minimum &mdash; close to a theoretical minimum, even. When a software application wants to send a memory buffer to a remote peer, it just writes a few bytes to device memory, and the network takes care of the rest. Incoming data is received just as simply, by polling device memory at the software's discretion. With RDMA, some transfers don't require any software interaction at all!

In theory, reducing software overhead doesn't just free up CPU time to be used by other parts of the software &mdash; it also decouples the CPU from the networking hardware, potentially allowing both to scale independently of one another.

Of course, there some drawbacks to this scheme &mdash; we'll get into those later &mdash; but for the right use cases, InfiniBand has proven to be a useful tool for achieving better network scalability.

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

