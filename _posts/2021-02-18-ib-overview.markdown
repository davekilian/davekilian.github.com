---
layout: post
title: InfiniBand
author: Dave
draft: true
---

This is the second post in a series on RDMA. In this post, we're going to cover some of the technical basics underpinning InfiniBand, the first technology to prominently feature RDMA.

If you haven't already, you may want to start with [the first post in the series](/rdma-overview.html) and come back here once you're done.

## InfiniBand

> This is a draft based on things I remember off hand, but history seems like something easily verified with research, so go do some research and revise as needed
>
> InfiniBand first came around in a time where we were tapping out Ethernet and PCI &mdash; we had devices and applications which used all available bandwidth and wanted even more &mdash; so we were looking for a replacement
>
> The dream of InfiniBand was to generalize all computer interconnects: why not replace disparate technologies like Ethernet and PCI with a single interconnect &mdash; InfiniBand &mdash; that works inside and in between computers? We can even replace some exotic technologies, like Fibre Channel, which already bends the inter-computer/intra-computer distinction anyways.
>
> That's how you get RDMA: clearly we want to support DMA inside a computer, so why not support an inter-computer DMA flow? That's how you end up RDMA.
>
> The architecture: everything new from scratch with a few on future scalability. New wiring specifications, signaling protocols, bus semantics and application interfaces. A whole new stack, from scratch, meant to replace a whole bunch of unrelated specs that have to interoperate today.
>
> Well, we live in the future now, and we know InfiniBand wasn't a big winner. It turned all-new everything was too much to stomach, even if it meant replacing a bunch of technologies with just one. We ended up using evolutions of our existing technologies, like PCI-Express for PCI and 10GbE for Ethernet.
>
> But there was one group of people who were really fascinated with InfiniBand: the high performance computing community, a.k.a. the folks that build supercomputers. Supercomputer builders generally  need to network together a lot of high-end computers, have a huge budget to work with, and often start with zilch, needing to purchase everything from scratch. So all technologies are in play &mdash; even ones that don't interop with existing commodity hardware/software &mdash; as long as they significantly improve the bottom line.
>
> The HPC group really likes InfiniBand for supercomputer networks because InfiniBand is designed around keeping the CPU out of the loop in network operations. This not only removes the CPU as a potential bottleneck for networking, thereby allowing the network to move more data faster, but also frees up the CPU to compute stuff, which is the whole reason you're building a supercomputer in the first place. Remember that a network with $N$ computers has $O(N^2)$ possible network connections, so as your computer scales linearly in size, the network utilization may scale super linearly! This makes it critical to manage the cost of networking overhead.

## InfiniBand Networking

> What makes InfiniBand's architecture different from conventional networks, and how do these differences help increase bandwidth while offloading networking tasks to free up the CPU?
>
> Remember that the goal of InfiniBand, the technologies that were fed into its creation, is to unify the interconnects inside your computer, which connect your hardware peripherals together, and networking interconnects that link computers together. If you look it at this way, conventional networking kind of works like programmed I/O: the CPU is orchestrating a potentially large number of small hardware data transfers, getting frequently interrupted every time a small transfer competes so it can start the next one. We'd like to make it possible for the CPU to set up these kind of large transfers once, and only get interrupted (called back) when the whole transfer has completed. In other words, the main goal is build up a network that supports DMA across machines.
>
> Here's the strategy the InfiniBand spec chose for offloading networking and minimizing CPU involvement. We'll explore each of these steps in more detail as part of this blog:
>
> 1. Build a network that never loses packets, so that hardware never has to deal with packet loss
> 2. Build sequencing natively into the network, which is easy to do once there's no packet loss
> 3. Expose hardware command queues to the CPU, which is easy given the network is sequential
> 4. Add DMA-style read/write operations built on top of this sequenced, lossless network
>
> Let's go explore how all of this works, and what this ends up looking like to the applications running on top of this network

## Lossless Networking

> A key goal with InfiniBand is to offload network transfers and coordination to hardware, so that the CPU only needs to be notified once transfers have completed. The challenge in doing this is that digital circuitry gets infeasibly complex quickly compared to software, so all protocols and algorithms we intend to implement in hardware must be conceptually simple. There's one thing that makes network transfers conceptually complex: packet loss.
>
> Recap why conventional/non-RDMA networks are designed with the assumption of packet loss and the need for technologies like TCP to provide retries, ordering and congestion control.
>
> All of this hopefully sounds exceptionally difficult to implement in hardware. The InfiniBand folks chose to sidestep this problem entirely, by designing a network that doesn't lose packets in the first place. [cue Eddie Murphy meme] If the network never needs to drop packets, then no hardware needs to deal with the contingency of lost packets, and pushing processing down to hardware circuits starts to look more feasible!
>
> There are a number of ways to implement a network that doesn't lose packets. The key observation is congestion is a problem of coordination, and we can avoid packet loss due to network congestion as long as every peer and every switch cooperates to ensure the network is never congested. This holds on networks that are owned and operated by a single entity &mdash; but this apporach doesn't work as well over the Internet, where it's not feasible (or practical) to expect so many users to cooperate.
>
> A simple scheme for lossless networking might use buffer reservations: when peer A wants to connect to peer B, both peers as well as all network switches together negotiate a single path through the network, as well as a buffer size in bytes that will be reserved along every hop of that path. As long as the sender never puts more data on the wire than can be stored in intermediate buffers, you guarantee there's always enough buffer along the connection to accept all the data you put on the wire. Nobody ever gets overloaded, so nothing ever gets dropped!
>
> It's worth noting, however, that this negotiation step may itself be expensive and require a bunch of transmissions between switches and peers. With InfiniBand, setting up and tearing down connections tends to be slow compared to conventional networking, but once you have a connection it's generally faster and higher bandwidth than conventional networking on a similar class of hardware.
>
> There are more advanced schemes that provide losslessness without explicit buffer reservations, such as adding a 'backpressure' mechanism for switch to ask everyone to stop sending it data for a while so it can drain its buffers. Links to ECN or something, if it's not too early.

## Network Sequencing

> In the section above, we saw a couple ways to create a lossless network, where all transmissions for a single connection between peers travel over the same network path, and no transmissions are ever lost.
>
> One interesting side effect of this design is that connections are, natively, sequential.
>
> Reminder of what TCP provides as a stream-oriented protocol that runs over a datagram-oriented protocol.
>
> With InfiniBand, transmissions are already sequential, because of the way we set up network. Peers using the network don't have to do anything special to ensure stream-oriented transmissions work.
>
> Describe sequence numbering scheme used to detect transmission errors. The sender maintains a counter, incrementing it on every transmission and including the counter value in the transmission. The receive maintains its own counter, incremeinting upon receipt of each transmission and checking it matches the value in the packet.
>
> The key difference is what happens when the counter doesn't match &mdash; with TCP, we need to buffer things out of order and wait for retransmissions to arrive, because a sequence number mismatch is expected. With InfiniBand, it 'should' be impossible for the network to deliver packets out of order. So what do we conclude? Something is wrong with the network, of course &mdash; so we simply terminate the connection. 
>
> Does it always work exactly like this? Maybe, maybe not. The point is to observe that all transmissions along a connection are strongly ordered &mdash; the receiving network card processes incoming transmissions in the exact order they were issued by the sending network card. The higher layers of the stack will take advantage of this.

## Command Queuing

> We now have a way for network cards to talk to each other over a lossless, strictly ordered networking fabric, but how does our code access the networking card to begin with?
>
> Remember that one of our goals is to offload network processing, so that the CPU just tells the network card about transfers it wants to do and later gets a notification when the transfer completes. In InfiniBand, these commands and notifications are exposed via a system of queues.
>
> Each connection to a remote peer is exposed locally as two queues:
>
> * A **Send Queue** (**SQ**) to which software enqueues work it needs to do
> * A **Receive Queue** (**RQ**) from which software dequeues incoming work
>
> Each element on one of these queues, called a **Work Queue Element** (**WQE**, which is sometimes jokingly pronounced "[Wookiee](https://en.wikipedia.org/wiki/Wookiee)"), generally maps to some sort of transmission sent over that network. A WQE on a send queue is often called a **Send Queue Element** (**SQE**) and a WQE on a receive queue is often called a **Receive Queue Element** (**RQE**). It's all very logical :-)
>
> Together, the send queue and receive queue for a connection are called a **Queue Pair** (**QP**). QPs are a core component of the InfiniBand spec &mdash; so much so that the term QP is colloqially used to mean 'connection to a remote peer.' Note that a queue pair refers to both local queues for a connection, not the corresponding endpoints between peers. (A diagram would help here &mdash; show two peers which each has a local send queue connected to a remote CQ, and circle each peer's queue pair and label them.)
>
> Why use queues to expose network cards to applications?
>
> First, because this allows applications to control the exact order in which data will be transmitted over the network. Remember that the fabric is lossless and strongly ordered &mdash; the order in which work elements are enqueued to the send queue is the order in which any corresponding transmissions will be carried out over the wire.
>
> Say for example that a peer enqueues three work elements to its local send queue: $S_1$, $S_2$ and $S_3$. These elements will be processed by the network card in the exact order the elements were enqueued (1-2-3). If the commands require the network card to transmit data to the remote peer, the transmissions will be carried out in that order &mdash; so the transmission for $S_1$ will be carried out first, and only when that's done will the transmission for $S_2$ be started.
>
> The remote peer will receive each transmission in the order it arrives, which is guaranteed to be the same order as above &mdash; 1-2-3. Let's say each transmission results in a notification being enqueued to the local receive queue; then the final receive queue will be
>
> As we'll see pretty soon, this strong ordering guarantee will be key to how applications end up using InfiniBand to transfer data.
>
> Other than this ordering guarantee, another reason InfiniBand is built around these queues is to allow vendors the option of exposing hardware queue memory, to minimize the cost of interaction between the CPU and the network card. Remember our original goal was to keep the CPU out of the loop in networking &mdash; not only to free up the CPU to do other, more useful work, but also to prevent the CPU from becoming a bottleneck preventing us from reaching higher transmission speeds. We now have a fast, lossless, natively sequential networking fabric, but all of that will be for naught if the CPU burns lots of time interacting with the local networking card!
>
> To reduce the overhead of calling into the network card and receiving notifications, it's common for vendors to map the hardware queue memory directly into the CPU's memory address space, allowing the CPU to directly read/write hardware send/receive queue memory (usually through a vendor-provided shim that insulates the application from needing to deal with the vagaries of accessing device memory directly).
>
> In many cases, this overhead is further reduced by directly mapping hardware queue memory into a user mode process's virtual address space. This allows the user mode program to directly read/write hardware queue memory, without needing to make (CPU- and time-intensive) kernel syscalls like would normally be required.

## InfiniBand Verbs

> Let's review what we've built ourselves so far: we have a lossless networking fabric where all transmissions for a single point-to-point connection between peers are received in the exact order they are sent, with no packet loss or retransmission. This fabric is exposed to applications as queue pairs, where commands on the queues map to transmissions in hardware. Everything on a connection occurs in one global order: if three commands are emplaced to a send queue in a given order, they will be executed on the wire in that order, received by the remote peer in that order, and placed on the remote peer's receive queue in that order.
>
> It's time to figure out exactly what an application can do with all this stuff. And one of those 'things' had better be Remote DMA!
>
> InfiniBand verbs, the etymology of 'verb'
>
> Two groups of commonly used data transfer verbs: the send/receive group and the RDMA group
>
> Send/receive semantics
>
> RDMA and registration/invalidate semantics

## Verbs in Practice

> If you can, register big blocks of memory and use your own application-specific addressing scheme to treat memory as one big pool. That's what the InfiniBand verbs were designed to do anyways.
>
> What if you need to act more like a traditional RPC network? Say you have clients and servers, where clients initiate requests to servers and servers send responses. Is RDMA still useful here?
>
> We'll need to use send/receive to exchange requests and responses. However, we want to do any associated 'data transfers' using RDMA.
>
> If the client is downloading data from the server, this ends up being pretty simple &mdash; the client registers memory, and includes the memory token in a 'request' header which is sent to the server via the send verb. The server receives the command, processes it, and then issues an RDMA write followed by a send with the response data. On the client side, the write is guaranteed to have been carried out before the receive is processed, so when the client gets a receive notification, it knows the data is already in its buffer. (Yay!)
>
> What if the client needs to upload a data payload? Then things get interesting.
>
> One approach is to allocate a pre-register a server-side circular buffer. The registration is persistent throughout the lifetime of the connection, and the client receives the buffer's remote address on connection setup. Any time the client wants to upload data, it first issues an RDMA-write into the circular buffer, and then it sends a request header which (among other things) specifies where in the circular buffer the data was written.
>
> But what if that circular buffer fills up, or the server is talking to so many clients that maintaining these circular buffers isn't feasible? Then the question gets interesting.
>
> For small data transfers, one thing a client could do is send the data payload along with the request. This typically means the server will need to copy the data out of the 'receive' buffer and into an application-controlled buffer, and that copy takes CPU cycles and time. As such, this is typically only feasible for pretty small data payloads, on the order of a few dozen kilobytes maybe.
>
> For larger data transfers, the client will probably need to fall back to registering its upload buffer locally and sending a token to the remote server as part of the request. The server, upon receiving the client's request, will need to then turn around and issue an RDMA read to bring the payload into local memory. This requires an extra network round trip before the server can process the request, but has the advantage of allowing the server to read directly into an application-controlled buffer, avoiding the copy mentioned in the previous paragraph.
>
> In many cases, a hybrid approach makes sense &mdash; the client looks at the upload data size and based on that decides whether to include the payload with the request or register it for remote read by the server.

## So What is RDMA?

> Now let's revisit everything above, and answer for the final time, what's RDMA?
>
> Here's the idea: we want to DMA between computers. This means a CPU on one computer goes an enqueues a command to its local networking card saying, "hey, I want to (read|write) (N) bytes of this remote computer's starting at this remote memory address," and "call me back when it's done!" Everything about the transfer, will be implemented by the networking hardware itself &mdash; a technique sometimes called 'offloading' the network from the CPU.
>
> Finish me

## Modern RDMA

> So far we've been focusing on InfiniBand, which is relatively easy to describe beacuse it's one spec &mdash; a large one, no doubt, but at least a single, self-contained entity. InfiniBand is still alive and well in the HPC community, but there are also new technologies popping up that provide a similar approach to moving network processing off the CPU.
>
> Ethernet, Converged/DC Ethernet (ECN), and RoCE
>
> iWARP

## Additional Reading

> Link dump

