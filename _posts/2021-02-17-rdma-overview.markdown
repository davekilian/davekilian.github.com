---
layout: post
title: RDMA
author: Dave
draft: true
---

RDMA is a family of networking technologies popular for building high-performance networks for things like supercomputers or private clouds. I've been working with RDMA networks on the job for a while now, and I've never been able to find a good, straightforward lowdown, so here I'm writing one of my own.

## What is "RDMA?"

15 seconds on Google will tell you that RDMA stands for "Remote Direct Memory Access." That's a mouthful if I've ever heard one! Let's unpack this a little.

To get started, we need to know a thing or two about your computer's hardware internals. Quite a few things, actually. Let's dive in!

## Inside Your Computer

Your computer is a network of interconnected parts. The two core parts are your CPU and your RAM.

Your CPU is a circuit that implements an extremely simple programming language: the [machine code](https://en.wikipedia.org/wiki/Machine_code) defined as part of the CPU's "[instruction set architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture)" or ISA. Machine code is a sequence of numbers (called 'instructions') that each select a function implemented as a subcircuit of the CPU (e.g. 'add' or 'multiply' numbers). Your code is ultimately compiled to a seqeunce of these instructions which, together, do something interesting.

Your RAM ([Random Access Memory](https://en.wikipedia.org/wiki/Random-access_memory), often just called 'memory') is a circuit for storing binary numbers. Your program (sequence of instructions) is stored in RAM; the CPU reads instructions from RAM automatically as they are needed. Your CPU also supports instructions that allow you to read and write RAM, giving your program place to temporarily store the data it's working on.

Your CPU and RAM are co-dependent &mdash; neither works without the other &mdash; but once you put the two together, you have a full-fledged computer! Albeit, not a very useful one ...

A practical computer needs more parts: things that add additional functionality, like disks for data storage or network adapters for accessing the Internet, and of course we'll need ways for us humans to interact with the computer, with things like keyboards, displays, touchscreens, etc. All of these things are called [peripherals](https://en.wikipedia.org/wiki/Peripheral) because, however useful they may be, they're not part of the programmable 'core' of your computer, which is your CPU and RAM.

How do peripherals work? Each is different, but they all share one thing in common: they need some way to interact with programs running on your computer. For this reason, each peripheral includes a little circuitry which is connected to the CPU/RAM 'core' of your computer. This circuitry often looks a little like a rudimentary computer in its own right &mdash; but, unlike a full computer, this circuitry isn't programmable; it's just a means of driving the device.

So, putting it all together, your computer is a CPU (which executes sequences of instructions), RAM (which stores the instructions as well as temporary program data), and peripherals that each talk with the CPU/RAM 'core' of your computer. In some ways, your computer is actually like a little network of mini-computers!

Let's take a closer look at how this network of peripherals works ...

## I/O

The act of a computer program interacting with a peripheral is called [I/O](https://en.wikipedia.org/wiki/Input/output), which is an abbreviation for 'input/output'. There are a few ways programs commonly interact with peripherals:

* Get data from a device (e.g. check which keys on a keyboard are being pressed)
* Send data to a device (e.g. change the image being displayed on a screen)
* Submit commands to a device (e.g. eject the disc in the disk drive)
* Receive an event notification (e.g. the user tapped the screen)

Exactly how programs do all of these things is defined by whoever made the CPU; in other words, I/O is part of the [ISA](https://en.wikipedia.org/wiki/Instruction_set_architecture) mentioned earlier.

> In case you're interested in the nitty-gritty details:
>
> Today, most ISAs use some form of [memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) (which, by the way, has no relation to Unix memory-mapped files). The basic idea is there are more possible [memory addresses](https://en.wikipedia.org/wiki/Memory_address) than there are bytes of RAM, so some of the unused addresses can be repurposed to refer to memory onboard a peripheral device instead of RAM. Programs can then use the CPU's 'write memory' to send data and commands to a peripheral, and the 'read memory' instruction to get data and the results of completed commands. 
>
> One thing you can't implement with memory-mapping is a event notification mechanism; notifications are typically implemented using an unrelated CPU mechanism called an [interrupt](https://en.wikipedia.org/wiki/Interrupt).

No matter how your ISA's I/O mechanisms work, there's one tricky problem you need to tackle for high-speed devices like storage disks and network adapters: copying large memory buffers.

TODO expand on what we mean by large memory copies





---



TODO

* One tricky problem is efficiently handling large data buffers. Devices like high-speed disks and networks.
* What you don't want is code that manually copies between device buffers and RAM; programmed I/O is slow. Say why.
* What you do want: the device does this instead. DMA







---

> TODO something like this:
>
> * Fundamental goals: write data out, read data in, submit commands, receive notifications
> * Many schemes for doing this. Memory-mapped I/O is one.
> * Limitation: these are based on (slow) memory reads and writes, each of which is small
> * What about devices that need to transfer large buffers? High-speed disks and networks.
> * What you don't want: the CPU doing all of these steps, and why
> * What you do want: let the device do these steps
> * DMA
>
> Older discussion:
>
> In the existing draft we got pretty deep into this. How deep do we want to go? Is the fact that we're redirecting memory addresses really worthwhile to the reader?
>
> Also, I'm not even positive we need to get into memory-mapped I/O details at all. It seems like I may have accidentally started writing the corresonding primer chapter here.
>
> What *do* we need to say?
>
> We need to build up to the 'who copies the buffer' problem. Which means we need enough of an idea of basic mechanics to understand that someone needs to copy a buffer.
>
> Ideally, mentioning memory-mapped I/O is a side subject and not a core concept for this discussion.
>
> We may need to mention what kinds of things you want to accomplish, but hopefully we can sidestep. The basic things are
>
> * Write onboard memory (e.g. image to display on screen)
> * Read onboard memory (e.g. poll keyboard for input)
> * Submit a command (e.g. eject the disc)
> * Receive a callback (e.g. the user tapped the screen)
>
> What's the basic structure then?

---











---

TODO move this to where we start talking about I/O

A CPU has a little onboard storage for holding variables (which are called [registers](https://en.wikipedia.org/wiki/Processor_register) for historical reasons), but not enough to store all the data programs need. Plus, the CPU has no space on board to store the program itself! For that reason, your computer also has [Random Access Memory](https://en.wikipedia.org/wiki/Random-access_memory) (often shortened to "RAM" or just "memory.")

RAM is structured as an array of bytes. Each index in the array is called an [address](https://en.wikipedia.org/wiki/Memory_address). The CPU is directly connected to RAM, and is always tracking the address (array index) of the next instruction; when each instruction has been executed, the CPU's circuitry automatically fetches the next instruction from memory in order to know what to do next. CPUs also provide 'read memory' and 'write memory' instructions, so that programs can use memory as well.

---

Throughout the ages there have been many approaches to I/O, but the dominant approach today is a technique called [memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O).

Here's the idea: we want a way for code to submit commands to peripheral devices' onboard memory, and read results back from peripheral memory too. We already have instructions to read and write RAM, which is also memory. Why not make it so the read memory / write memory instructions can access both RAM, and peripheral device memory?

The key to making this work is there are more possible memory addresses (indices into the memory as a byte array) than there are bytes in RAM, so some addresses inevitably end up unused. We can repurpose some of the unused address to refer to peripheral device memory!

For example, say we had a primitive computer with a tiny amount of RAM (10,000 bytes), that uses 16-bit addressing (so all addresses from 0 to 65,535 are possible), and the only things we have connected are a keyboard and a display. We might set up memory mappings like this:

| Device   | Memory Size (in Bytes) | Mapped Addresses |
| -------- | ---------------------- | ---------------- |
| RAM      | 10,000                 | 0-10,000         |
| Keyboard | 2,000                  | 10,000-12,000    |
| Display  | 7,500                  | 12,000-19,500    |
| (unused) | ---                    | 19,500-65,536    |

Say you were writing code that needs to poll the keyboard to check whether a certain key is being pressed. (It's not common to write this code yourself nowadays, but your operating systems certainly has code to do something like this built-in.) This keyboard-polling code would do the following:

* Figure out what keyboard memory address stores the key's state; say, `1500`
* Figure out where in memory the keyboard is mapped: `10000` in the table above
* Add these two addresses together to get a final memory address: `11500`
* Issue a 'read memory' instruction to that address (`11500`)

From the programmer's perspective, this is all that's needed. The next instruction in the program can conclude that this read completed, and the value read from keyboard memory is now stored in a CPU register, ready to be operated on.

Physically, in hardware, we'll need to make this work somehow. We'll start by adding a new chip to broker access between the CPU, RAM and peripherals, like this:

> Diagram

This new chip's job is to broker access between the CPU and memory, decoding memory read/write addresses according to the table defined above. Continuing the keyboard example, let's say a memory read to address `11500` arrives at our new chip; what does the chip do?

* Consult the memory mapping table above
* Figure out which range the address `11500` falls into (`Keyboard`)
* Subtract the base address of that range (`11500-10000=1150`)
* Issue a read to `Keyboard` at address `1500`
* Forward the result back to the CPU to be stored in a register

How does the chip issue a read to the keyboard? In short, the chip acts as one of the devices on a hardware device called a [bus](https://en.wikipedia.org/wiki/Bus_(computing)), which is a set of physical and logical protocols for setting up transfers between peripherals. We're not going to get into the nitty-gritty details of buses today.

One more thing about this chip: it doesn't have any one standard name, but it exists in one form or another in just about every ISA in existence today. On desktop computers, this has classically been two chips working together &mdash; a [Northbridge](https://en.wikipedia.org/wiki/Northbridge_(computing)) that connects the CPU to RAM and the highest-speed peripherals, and a [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)) for lower-speed peripherals (Wikipedia has [a nice diagram](https://en.wikipedia.org/wiki/Southbridge_(computing)#/media/File:Motherboard_diagram.svg)). On today's higher-end desktops, the northbridge has been subsumed into the CPU itself for maximal bandwidth. And, on phones and other mobile devices, anything goes!

So far, we've been talking about moving small amounts of data around. What if we want to copy some big buffers? Say you wanted to read 8 KB from a disk &mdash; in the example above, this would seem like quite a lot, but it's actually a fairly small transfer for modern-day computers.

Here's how we'd make this transfer our memory mapping scheme:

> Diagram with numeric labels for each step below

1. The CPU writes a command to disk memory saying what to read
2. The disk reads the requested data into disk memory
3. The disk [interrupts](https://en.wikipedia.org/wiki/Interrupt) the CPU (like a callback) to indicate it's done
4. The CPU copies data from disk memory to main memory (RAM)

This scheme has a name &mdash; [programmed I/O](https://en.wikipedia.org/wiki/Programmed_input–output) &mdash; so we know it'll work. But there's a problem: it's so .. darn ... slowwwwww.

The key problem is that memory copy loop in step 4. This works by writing software which loops over each address in the disk memory buffer, first reading from disk memory into a CPU register, then writing from the CPU register into main memory. This potentially involves thousands of iterations, and while the loop is running the CPU is completely occupied and can't be used for anything else. On modern computers, the CPU is way faster than memory, which only compounds how much CPU time this copy loop wastes.

That's why modern I/O technologies employ a technique called [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA).

With DMA, the high-level steps are the same as above, with one key difference: the *disk* does the copy loop in step 4 instead of the CPU. This cuts the CPU free from the process, allowing it to get more done while this (potentially slow) memory transfer is ongoing. Nice!

So, to put it all together, a DMA-style transfer would look like this:

> Diagram with albels below

1. The CPU writes a command to disk memory saying what to read
2. The disk reads the requested data into disk memory
3. The disk copies data from disk memory into main memory (RAM)
4. The disk interrupts the CPU to indicate it's done

When DMA was first introduced, CPUs were typically much faster than peripheral devices and their corresponding memory transfers, and DMA was key to ensuring CPUs weren't being bottlenecked by peripherals. With some of today's highest-end hardware, the situation has been inverted: now we have some super fast peripherals that need DMA to avoid being bottlenecked by the CPU!

And with that, we *finally* have enough background information to get to the main topic of this blog ... 

## Remote DMA

Did the heading for this section give away where all this is headed?

You see, it's *technically* correct to expand "RDMA" fully to "Remote Direct Memory Access," and that's why you'll see many articles write it out that way. But it's arguably more enlightening to expand it as "Remote DMA," because the core idea of RDMA is to do a DMA transfer *between computers*.

On a 'conventional' networking technology like the kinds you used to retrieve this blog, data transfers work kind of like programmed I/O: software running on a CPU has to break down the buffer into messages on the sender side, and reconstruct them on the receiving end. This ties up the CPU on both ends of the network connection; wouldn't it be nice to cut the CPU out of the loop?

On an RDMA network, data transfers are still implemented by splitting the buffer into messages on one end and reconstructing the buffer from the received messages on the other end; but with RDMA, the  [network card](https://en.wikipedia.org/wiki/Network_interface_controller) handles this instead of the CPU. Like with DMA, the benefits to this are two-sided: the CPU is freed up to do other work, and the network can scale to higher speeds than a CPU could feasibly handle.

That all sounds great, right! Maybe a little too great to be true? If this is so much more efficient, why isn't it widespread?

The short answer is that managing buffer transfers in hardware *on a conventional network* is still something an open problem; RDMA is generally implemented by making fundamental changes to the network itself, to remove a bunch of tricky error cases and make hardware offload more feasible. Some of these changes require tight cooperation between peers: if even a single node misbehaves, it can bring down the entire network. There's no hope of achieveing this kind of tight cooperation over the Internet, so anything that connects to the Internet probably isn't using RDMA.

But, if you're building a large, high-performance computer network by yourself &mdash; maybe you're building a [supercomputer](https://en.wikipedia.org/wiki/Supercomputer) or your own private [cloud](https://en.wikipedia.org/wiki/Cloud_computing) &mdash; then using an RDMA-capable network technology is more feasible, and can potentially allow your system to scale better and run faster. That's why you still pretty much only hear about RDMA in the context of the high-performance computing (HPC) space, and not much else.

Sound good? Well, now you one *one* definition of RDMA. But there's a second definition as well!

You see, everything we've said so far is correct in the technical sense, and if you search for RDMA on the Internet, the above is more or less what you're going to learn. But the term "RDMA" is also commonly used as an umbrella term for InfiniBand, and InfiniBand-related technologies.

InfiniBand was the first prominent technology to feature RDMA. The designers were trying to create one unified interconnect (InfiniBand) which works both *between* computers, like a conventional network, as well as *inside* a computer, to link peripherals to CPU+RAM "programmable core." In trying to do this, they found they needed to a way to extend DMA to work across computers, and so "Remote DMA" was born.

Although InfiniBand never really took off widely, it did gain a strong hold in the HPC community, and gave rise to extensions and competing standards. This all happened organically, and the community never came up with a good umbrella term to describe all of this tech as a whole. It's also preferable to avoid saying "InfiniBand" in marketing materials, because "InfiniBand" is a trademarked term. (Please remember to include the implicit "(TM)" every time you read the word "InfiniBand" in this article.)

Without a better option, the de facto term for all these technologies ended up being "RDMA." Which means, depending on the context, "RDMA" could refer to "Remote DMA," the thing we've spent so much time describing so far, or it could mean "networking technology that features Remote DMA" (and is probably related to, or at least inspired by InfiniBand).

Not confusing at all, eh? Well, stick around for a while and you'll get used to it.

Anyways, now that we (finally) have the basic definitions down, let's take a look at RDMA in more detail. We'll do that by taking a closer look at InfiniBand, and its key architectural decisions. Most RDMA technologies have at least some roots in InfiniBand, so even if you have no interest in deploying InfiniBand, understanding it will probably help you understand whatever technology you do intend to deploy.

## InfiniBand

When looking at a technology, it's often useful to understand the historical context in which it was invented.

InfiniBand first came about around the turn of the milliennium. At the time, the dominant specification for connecting high-bandwidth hardware peripherals to a CPU was an interconnect called [PCI](https://en.wikipedia.org/wiki/Peripheral_Component_Interconnect); however, it was becoming clear that PCI's days were numbered, as the most bandwidth-hungry top-of-the-line peripherals around were starting to bump up against PCI's fundamental bandwidth limitations. The industry as a whole was starting to look for a modernized, higher-bandwidth alternative, and came up with several competing specifications to try and fill the niche.

Today, we know the winner of the race was [PCI Express](https://en.wikipedia.org/wiki/PCI_Express), an interconnect that is still the dominant protocol for the highest-speed peripherals 20 years later. But there was another notable contender in this race &mdash; InfiniBand &mdash; which may never have enjoyed widespread dominance, but did find itself a comfy niche in the high-performance computing space.

InfiniBand itself was the merger of two competing standards &mdash; one called "Next Generation I/O" and another called "Future I/O" &mdash; and was heavily inspired by another specification called the [Virtual Interface Architecture](https://en.wikipedia.org/wiki/Virtual_Interface_Architecture) (VIA). The designers of all these specifications noticed the growing similarities between interconnects (which link peripherals inside a computer) and networks (which link computers), and were trying to build a single technology to serve both as a hardware interconnect and a network.

("Huh?" you might ask, "Why would you want to unify interconnects and networks?" Remember these specs were trying at least in part to cater to people building supercomputers, and with supercomputers the distinction between 'different computers working together' and 'different parts of the same computer' can get blurry. If you're interested in an example, consider reading about [Storage Area Networks](https://en.wikipedia.org/wiki/Storage_area_network).)

In the end, InfiniBand ended up resolving some of the design differences between interconnects and networks in favor of interconnects (i.e. making the network work more like an interconnect). In several key ways, the spec revolves around RDMA, and bending the design of the network itself to fit the requirements of implementing RDMA efficiently in hardware. You could summarize these changes as follows:

1. Build a sequential network that never loses packets, to make hardware offload easier
2. Expose hardware command queues to the CPU, ordered according to network sequencing
3. Add DMA-style read/write operations built on top of this sequential, lossless network

Let's start at the bottom level in the technology stack our way up: what does it mean for a network to be 'sequential' and 'lossless,' why does it matter, and how do you implemet something like that?

## Sequential, Lossless Networking

> Networks fundamentally work by connecting computers together and copying messages between them so they arrive at their network. Special-purpose computers called 'switches' are responsible for bridging multiple of these point-to-point connections together.
>
> Conventional networks don't place a lot of restrictions on what these switches can and can't do. Here are some things that commonly occur in conventional networks (and on the Internet as a whole):
>
> Packet loss: some transmissions never make it to the intended recipient. Interestingly, it isn't always an all-or-nothing affair: you can send 100 transmissions to a single recipient and the recipient might lose a random subset of those messages
>
> One common reason this happens is congestion. If you look at a switch, you'll see it's possible for multiple input lines to all try to send to a single output line, which can overwhelm it. The typical way switches handle this is to randomly pick all the transmissions it can deliver in time and drop the rest.
>
> Because of packet loss, and for other, more esoteric reasons, it's completely possible for 
>
> Handling conditions like packet loss and out-of-order delivery typically requires complex software code heavily laden with conditions and tuneable policies; this is the kind of thing that doesn't work very well in hardware, the circuits just end up becoming infeasibly large and complex. So instead, InfiniBand does away with both!
>
> Here's an example scheme that accomplishes this:
>
> The basic idea is to use buffer reservations. When peer A wants to connect to peer B, both peers as well as all network switches together negotiate a single path through the network, as well as a buffer size in bytes that will be reserved along every hop of that path. As long as the sender never puts more data on the wire than can be stored in intermediate buffers, you guarantee there's always enough buffer along the connection to accept all the data you put on the wire. Nobody ever gets overloaded, so nothing ever gets dropped!
>
> It's worth noting, however, that this negotiation step may itself be expensive and require a bunch of transmissions between switches and peers. With InfiniBand, setting up and tearing down connections tends to be slow compared to conventional networking, but once you have a connection it's generally faster and higher bandwidth than conventional networking on a similar class of hardware.
>
> There are more advanced schemes that provide losslessness without explicit buffer reservations, such as adding a 'backpressure' mechanism for switch to ask everyone to stop sending it data for a while so it can drain its buffers. Links to ECN or something, if it's not too early.
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

