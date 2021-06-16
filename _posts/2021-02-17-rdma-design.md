---
layout: post
title: Let's Design RDMA
author: Dave
draft: true
---

In the [previous post](TODO), we talked a little about RDMA, its history, what it's used for, and some of the key terminology. In this post, we're going to do a deep dive into how it works, with a focus on why it was designed to work the way it does.

My favorite way to learn a system design is to walk the process of designing the system: I like to try and understand what the designers already had and what they were trying to do, and then see what problems popped up along the way and how they went about solving them. In this post, we're going to do just that, for RDMA.

Initially, we'll focus on InfiniBand, not only because it's historically the first major RDMA technology, but also because it lays the groundwork for other technologies that came afterward. After we understand InfiniBand, we'll take a look at some other RDMA technologies and see how they relate back to the original InfinIBand specification.

But first things first: a quick review of conventional, non-RDMA networking is in order.

## Conventional Networking

If we want to understand what InfiniBand's designers were doing, it helps to first understand how networks looked before they started to draw up their designs. It turns out the conventional networking stack of that time was pretty much the same as it is today: Ethernet, IP, TCP and BSD sockets. So let's do a quick review.

These are big topics, and we'll keep our discussion brief. That said, if you feel confident that you don't need a refresher on any of these topics, you can skip forward to the next top-level heading. Otherwise, let's start at the bottommost layer &mdash; Ethernet &mdash; and work our way up.

### Ethernet

Ethernet is a hardware + software specification that implements computer networking from scratch, down to the level of wire gauges and signaling voltages. Ethernet has been around for nearly 40 years, and has been continually evolving since its inception.

Ethernet was originally envisioned in the 80s as a 'shared media' system. This just means that all the computers in an early-80s Ethernet network were directly wired together, using a system of coaxial cables and splitters not unlike what we still use for cable TV nowadays. With shared media, only one computer can transmit at a time across the whole network, and every computer receives every transmission.

Much of the engineering behind Ethernet went into figuring who 'gets to' transmit at any given time. This is done by assembling a small transmission, called a 'frame,' transmitting the frame on the network, and then using a [somewhat arcane set of heuristics](https://en.wikipedia.org/wiki/Carrier-sense_multiple_access_with_collision_detection) to detect if that transmission might have overlapped with someone else's transmission, and back off / retry if so. These collision avoidance heuristics are set up so that, if everyone operates correctly, then every transmitter will eventually get a chance to transmit each frame.

As computers grew ever more sophisticated and hardware costs plummeted throughout the 90s and early 2000s, shared media Ethernet all but disappeared. Today, Ethernet networks rely on dedicated networking devices called [switches](https://en.wikipedia.org/wiki/Network_switch). The Ethernet ports on a switch are connected to a central processing core which has a single job: forward each frame to its intended recipient.

To understand how switches, work, let's look at a concrete example. Say we have a switch with four ports connected to four computers: port 1 is connected to computer $A$, port 2 to computer $B$, 3 to $C$ and 4 to $D$. Now, say $A$ wants to send a frame to $B$. It assembles an Ethernet frame with a 'To:' address of $B$ and sends that frame to the switch, which receives the frame on port 1. The switch sees to 'To:' address is $B$, and knows $B$ is connected on port 2, so it sends the frame out on port 2. $B$ receives this frame and can now process it.

Interestingly, while $A$ is transmitting a frame to $B$, it's also perfectly fine for another computer (let's say $C$) to also transmit to $B$ in parallel. The switch decides which frame ($A$'s or $C$'s) gets transmitted to $B$ first, and which must wait in a temporary memory buffer. This eliminates the need for computers on an Ethernet network to handle transmission collisions themselves. Switches also allow for greater parallelism than shared media: for example, $A$ can transmit to $B$ in parallel with $D$ transmitting to $C$ on the same switch, and nobody has to wait for anybody else.

> Even though collisions in Ethernet networking are mostly a thing of the past, the engineering that went into these lives on in wireless networking technologies, many of which are just shared-media Ethernet running on open-air radio waves instead of a system of interconnected coax cables.

Modern Ethernet networks are built by connecting computers together using switches, and then progressively joining those switches together until every computer can reach every other computer.

### IP

The Internet Protocol (IP) is a software protocol for bridging networks (such as Ethernet networks) together. Before IP was invented, we already had networking technologies sort of like Ethernet (although, technically, Ethernet hadn't been inveted yet). But networks back then were usually small and there weren't many good ways to connect them together. If you wanted to connect to a network you weren't physically wired to, you usually had to dial in using a device called a [modem](https://en.wikipedia.org/wiki/Modem); if you wanted to use two different networks, you either had to disconnect from one and dial into another, or you needed one computer hooked up to two phone lines (expensive!)

The Internet Protocol was invented as a way of building 'internets' that link these networks together. An internet is a virtual network overlaid on top of two or more 'real' networks running something like Ethernet. Each of the constitutent networks that makes up an internet is called an 'autonomous system' (AS), and two autonomous systems that have been connected to one another are called 'peers.' An internet is formed out of peered autonomous systems, such that each autonomous system is connected to each other autonomous system through at least one of its peers.

The Internet Protocol imposes a little bit of structure on top of this arrangement. Each computer that can use the resulting internet has a unique 'IP address,' in addition to its 'real' address according to the underlying network technology. (So for an autonomous system that runs Ethernet, each computer would have both an IP address and an Ethernet [MAC address](https://en.wikipedia.org/wiki/MAC_address).) IP transmissions are called 'packets,' and each packet must fit in a single underlying network transmission &mdash; so when you run IP on an Ethernet network, each IP packet must fit inside one Ethernet frame.

Originally, internetworking was envisioned as a management tool for large networks; for example, research schools were interested in linking their networks together to help researchers in different instituations work together. But then one thing led to another, [six degrees of separation](https://en.wikipedia.org/wiki/Six_degrees_of_separation) kicked in, and soon enough all the internets of any note peered together, leaving us with the one giant internet we today call the "Internet" (with a capital 'I').

Interestingly, IP is now so ubiquitous that it's commonly used even in private networks that will never internetwork or connect to the Internet. That's because key pieces of networking technology are built on top of IP, and not having IP means you lose out on those technologies as well. One notable example of such a technology is TCP.

### TCP

TCP exists because the technologies that IP runs on &mdash; including Ethernet, notably &mdash; allow for packet loss by design. This is mainly because of a situation called *congestion*.

Think back to that 4-port Ethernet switch we were talking about earlier: we said that if $A$ and $C$ each try to send a frame to $B$, then the switch will arbitrarily pick one to transmit right away and buffer the other to be transmitted next. That's fine, but what if this situation keeps up indefinitely? Say we're running on Gigabit Ethernet &mdash; that means that each network link can transmit 1 gigabit (one billion bits) every second. That might sound like a lot, but remember the $A$ and $C$ can *each* send 1 billion bits of data to the switch every second, and all of it could be addressed to $B$. If the switch is getting 2 billion bits of data for $B$ every second, but the link can only transmit 1 billion per second, what's the switch going to do? It can't buffer frames forever!

This situation, where the incoming data rate for a link is higher than what the link can transmit, is called *congestion*. Once a link is congested, there isn't anything particularly smart a switch can do: it'll transmit as much as it can and forget about the rest. (In this example, 'the rest' is about half of the incoming data!) This means that neither Ethernet nor IP is *reliable*: IP packets / Ethernet frames can get lost on their way from the sender to the recipient, and that's expected due to the way both protocols handle congestion.

Anyone who tried building nontrivial networking applications on top of these systems would have quickly realized that handling packet loss is cumbersome, to say the least! That's why the Internet Protocol is almost always used with the Transmission Control Protocol, or TCP.

TCP is a protocol for sending and receiving IP packets, automatically detecting and handling packet loss. The idea is for the transmitter to build its own local model of the recipient's state, using timeouts and other heuristics to detect that packets may have been lost, and retransmitting them as appropriate. The recipient cooperates by sending information about its local state back to the transmitter, buffering packets received out of order, and ignoring duplicates. The details are somewhat complex, but the complexity is hidden away from the applications using TCP. Applications see a TCP connection as a reliable stream of bits, where data appears on the recipient side in the exact order it was sent by the transmitter. Simple!

TCP is pretty much always implemented as software code. It was once common to see applications deployed with their own TCP implementation, but nowadays most applications use the TCP implementation included with the operating system, be it Windows, Linux, BSD, etc.

### Sockets

Modern computers run lots of programs, and each of these programs need to share the hardware. Your operating system provides interfaces to those programs that arbitrate access to the hardware; this is done not only to ensure each program gets its fair share of the hardware resources, but also to isolate programs from one another, so that a buggy program can't cause problems for other programs. (Think otherwise how hard it would be to debug stuff!)

For network hardware, pretty much every operating system uses some variant of the Berekely sockets API, which is a C interface that debuted as part of the Berkeley Software Distribution (BSD) operating system (a variant of Unix that predates Linux).

If you've written code that uses the network before, chances are you're already familiar the idea of sockets, connections, streams and so on. If not, [beej's guide](https://beej.us/guide/bgnet/) is a good place to start if you're interested. You don't need to know the details of the Berkeley sockets API to follow the rest of this document, however. The important thing to know about the sockets API is that operating system implements it by interacting with the network card on your behalf &mdash; and that the operating system makes sure multiple programs can use the same network card without the programs knowing about each other.

### Message-Passing

One final, but important side-note has to do with how programs tend to use TCP sockets in practice.

Applications frequently use networks to send what we'll call 'messages.' A message can be anything: a JSON or XML string, a byte array, a serialized data structure, you name it! The important thing about a 'message' in this context is that the application will send multiple such messages to the receiver, and each message is standalone &mdash; each can be parsed by itself.

Using TCP to to implement a message-passing system like this is tricky. TCP gives applications an 'infinite pipe' with the guarantee that, whatever order bytes go in on the sender side, is the order they'll come out on the receiver side. But TCP doesn't understand what it's sending and receiving, so there's nothing stopping the receiver from partially receiving messages. This means the receiver needs code &mdash; often application-specific custom code &mdash; to add another layer of buffering for messages that have been partially delivered by TCP.

For example, say you have a TCP connection, where $A$ wants to send $B$ some messages: a 100-byte message followed by a 200-byte message. $A$ calls `send()` twice: first with the 100-byte message, then with the 200-byte message. What's going to happen on $B$ when it calls `recv()`? There are lots of possibilities:

* Maybe $B$ gets a `recv()` with the 100 byte message, then another with the 200 byte message
* Or, maybe a single `recv()` call returns 300 bytes
* Or, maybe `recv()` only returns 50 bytes at a time (so it takes 6 calls to get all the messages)
* Or, maybe `recv()` returns 150 bytes each time

Applications often end up having to deal with this using custom 'message framing' protocols. This means the sender has to prepend a little 'header' saying 'the following message is $N$ bytes' before then immediately sending the message; the receiver has to parse that header to know how much more data is in the receive stream before the message can be parsed, and the next 'header' can be found.

This adds a degree of complexity for application developers, but networking technologies have clearly been successful nonetheless. We'll come back to message passing later on in this document, so keep this in the back of your mind.

## InfiniBand

The entire Ethernet / IP / TCP / BSD sockets stack outlined above already existed and was widely popular when InfiniBand came about in the early 2000s. Ethernet switches were already cheap and ubiquitous, and things like shared media Ethernet and repeater hubs were either gone or on their way out. People typically deployed IP on top of Ethernet, and wrote application using TCP sockets on top of this network. All in all, the stack already looked very similar to the way it still does today.

So, then why did InfiniBand come about? What did we need that the conventional stack didn't give us?

### Design Goals

> The goal is to maximize performance in large, private networks. Think of things like data centers and super computers. Performance means maximizing throughput and minimizing latency. Any time software needs to get involved, performance suffers: just the overhead of handing off between the network card and the CPU is already a limiting factor, no matter how well the software is written. Conclusion: we want to decouple the network from software, by having no software involved in network transfers.
>
> So, how are we going to do this? We’re going to attack these primary sources of software overhead:
>
> - Software-assisted packet ordering, retransmission
> - Software-assisted message framing
> - Software-assisted virtualization of the network card
> - Software-assisted memory management and buffer copies
>
> We fundamentally have two tools: we can push this stuff down to the hardware level, so that some combination of the network card and network switches does it for us; or, we can rearrange the network such that the whole problem ‘goes away.’ Turns out we’ll do a little of both. 

### Reordering and Retransmission

> Start with a concrete example of why congestion occurs even in a well designed modern switched network. There’s exactly as much bandwidth going in and out of each connection, and if bandwidth in > bandwidth out for a particular link, you’re screwed.
>
> Okay, so if congestion is unavoidable, but we don’t want software that deals with congestion, I guess we’ll do it in hardware, on the network card? Anyone with experience designing hardware will let you know that’s challenging: a typical TCP implementation builds a model of the remote peers state and uses a lot of state transitions, heuristics, if-this-then-that, all of which would result in very large and complex circuits.
>
> Okay, so let’s try it the other way: can we rearrange the network so the problem ‘goes away?’ What if all peers on the network cooperated to make sure no link gets overloaded?
>
> It turns out that’s relatively straightforward! All you need is to add some control plane logic in both the network card and the switch. Here are a few schemes you could try:
>
> * Bandwidth reservations
> * Credit systems
> * Feedback / ECN
>
> A real, robust scheme would probably use a combination of these
>
> Talk about IB’s system. Maybe some research require here.
>
> That gives you protection from loss. What about reordering? In the absence of loss, the only way to reorder is if two packets take two different paths through the network and thus ‘race.’ Solution: every connection must only use one path through the network. We’ll negotiate this path on connection setup, which might be necessary for congestion control anyways.
>
> The result is sometimes called a ‘lossless fabric.’ Connections on IB will be all-or-nothing: peers can assume no packets are ever lost, and if a packet is ever lost the network will treat it as a connection-fatal error and explicitly tear down all state (peers and intermediate switches).
>
> Describe the per-connection sequence numbering scheme used to detect errors and destroy connections. This is a cheap and effective way to make sure there aren’t errors; you need the more complex schemes above to prevent congestion-based errors in the first place.

### Message Framing

> Many systems need to put a bunch of unrelated ‘messages’ on a wire to a peer. Some examples, maybe a multiplayer game or a web server? 
>
> The conventional tools aren’t great for this. You can use UDP to directly construct IP packets, but then you have to deal with loss, and size is very limited (~1KB usually). TCP isn’t a perfect fit either, because it provides one endless stream of data with no demarcation.
>
> Can we do better, with our lossless fabric? Yup! It’s pretty easy to provide a high-level message passing interface once you have a lossless connection set up.
>
> Describe a basic scheme. Don’t spend too much time here because it’s not that complex.

### Virtualizing the Network Hardware

> Nowadays when you talk about ‘virtualization’ you think about virtual machines … but the real OG of virtualization is the operating system. Rehash remzi 
>
> The operating system is software; can we set up an interface that provides the same protection, but doesn’t go through an operating system?
>
> Well, we’ll have to see what interface the hardware exposes to the operating system, and figure out how to expose that to a user mode process in a ‘safe’ way
>
> Quick overview of memory mapped IO and interrupts. Do an anatomy of a single IO just to give an idea of how this would work.
>
> So, how do we expose this to user mode processes? There’s two problems: memory mappings and interrupts.
>
> We could map device memory in user mode, but then we have a situation where processes need to know about each other, and a bug in one process could manifest in another process.
>
> What if we had a per-connection memory mapping? Then the user mode process that created the connection can access the associated device memory for its own connection, but not for other processes’ connections. Hardware-assisted virtualization.
>
> What about interrupts? There isn’t really a way to bypass the kernel for interrupts - at least not in the way operating systems usually work. Let’s work around it by trying to avoid interrupts altogether if we can - they’re inefficient at the hardware level anyways. Batching and coalescing.
>
> By the way, wanting to avoid interrupts goes both ways - whatever we do to submit work to the network card can itself by slow, so best if we can batch:coalesce commands too.
>
> So what we want is a per connection device memory mapping that supports both send-side and receive-side batching and coalescing.
>
> Enter, the queue pair. Explain the interface. You get: direct access to device, batching, control over network order. Each QP mapping only provides access to your connection. 

### Zero-Copy Networking

> So now we have this zero-overhead message-passing interface on top of a lossless fabric. You might think mission accomplished, but there’s one more subtle problem left behind: buffer management.
>
> Meta: I’m on the fence on how to introduce this. One tack we could take is the idea of buffer management: having to receive into an arbitrary buffer means a local copy from ‘network’ buffer to ‘application’ buffer. But that only covers RDMA-read and isn’t even a very interesting use case. Another tack is to point out the DMA/RDMA parallels, but that’s a weird reason to develop RDMA semantics anyways. How can we arrive at RDMA at a basic, intuitive level?
>
> One thing we could try is say that many network operations are really remote memory operations, in that you want someone else to read from or write a specific buffer on your computer?
>
> Or we could start with the ‘write into this buffer’ receive scenario, come up with RDMA, and then say hey, why not let remote users read too?
>
> Here’s another stab: the core problem is “I want to request that data into this buffer” and not have any intermediate copies or software interaction along the way. There’s actually two ways to do that: remotely read that data into a local buffer, or have the remote peer remotely write the data into a local buffer. In other words, it’s a memory copy across the network. RDMA. The core idea is to be able to remotely identify a resource such that the network card can independently read or write the data, without any software getting involved
>
> So, okay, here’s what I think the rub is: to see why RDMA semantics reduce overhead over send/receive, you need to understand the in-practice post already, which is a circular dependency. So I think we will have to handwave a little and then forward link to the in practice posts if you wanna see how these get used to reduce overhead in various scenarios 

### In Review



## Beyond InfiniBand

> A big chunk of the work being done in the RDMA space is creating hybrids between IB and conventional networks. IB has zero built in interop with conventional equipment but conventional equipment is desirable

### RoCE

> RDMA over Converged Ethernet. Pronounced “rocky.” The goal is to provide the IB API interface on top of an Ethernet network. Can we do it?
>
> One of IB special sauce is a system where all nodes of the network cooperate to ensure no packet loss. Ethernet does not provide these guarantees, so what is one to do?
>
> What if we add those guarantees to Ethernet.
>
> ECN/pause/xon/xoff as Converged Ethernet (where does “converged” come from?)
>
> Once you have lossless Ethernet you can totally bolt the top half of IB on top. That’s RoCE.
>
> RoCE 1 uses raw Ethernet frames to carry IB data and metadata. RoCE 2 uses Ethernet frames carrying IP packets which in turn carry IB stuff. 
>
> That’s basically it.

### iWARP

> Meta: I actually have some reading to do here! IIRC the goal is to embed an IB-like protocol in a TCP stream and provide hardware offload and RDMA that way. What you have to do, then, is the thing we originally said is hard: do TCP in hardware
>
> RFC 5040 defines a data placement system
>
> 5041 discusses specifics of integrating with TCP, supposedly
>
> I suspect the idea is you can do TCP but in a way that each packet tells the receiver where the data belongs. Even if you get data out of order you can still figure out how to RDMA it

## Design Critique

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
