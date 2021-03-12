---
layout: post
title: RDMA
author: Dave
draft: true
---

"*Everything you wanted to know about RDMA (but were too afraid to ask)*"

Since you've ended up here, I'm guessing you're looking into RDMA, a networking technology popular for building large, high-performance private networks for things like supercomputers or private clouds. I've been working with RDMA networks on the job for a while now, and I've never been able to find a good, straightforward lowdown on RDMA, so here I'm writing one of my own.

As a software developer by trade, I'm writing this guide with other software developers in mind. My goal is for you to walk away with an idea of how programming against RDMA networks is different from conventional Ethernet/IP/TCP networking, with a general idea what's going on in the network hardware and why the technologies are different.

Let's get started with a little terminology:

## What is "RDMA?"

15 seconds on Google will tell you that RDMA stands for "Remote Direct Memory Access." That's a mouthful if I've ever heard one! Let's unpack this a little.

To get started, we need to know a thing or two about your computer's hardware internals. Quite a few things, actually. Let's dive in!

## Inside Your Computer

Here's a basic schematic of the inside of your computer.

> TODO

As we can see, your computer is a network of interconnected parts. Let's start by taking a look at the core parts of your computer: your CPU and your RAM.

The CPU is an electric circuit that implements a very basic programming language called 'machine code.' Usually, this language and the CPU itself are co-developed, and together, these two things are called the CPU's "[instruction set architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture)" or ISA. (As you can already see, hardware people love three-letter acronyms, i.e. TLAs).

To vastly oversimplify, the heart of the CPU is a bunch of little calculator circuits &mdash; an add circuit, a multiply circuit, a divide circuit, and so on &mdash; plus a 'circuit picker.' Machine code consists of a sequence of instructions, each of which is a number telling that 'picker' which circuit to pick next. As a software developer, your job is to line up these instructions strategically so that, together, they do 'something.' For example, here's a program (sequence of CPU instructions) which together calculate the 4h Fibonacci number:

>  TODO show a basic x86 fibonacci with each instruction annotated

As we saw in the example above, your CPU has circuitry to store a few variables for intermediate values you're not done working on. For historical reasons, these variables are called "registers." It's only cost-effective for a CPU to have a handful, and programs often need to store more state than the CPU has registers. That's why your CPU is connected to RAM: RAM provides a place to offload program variables that you don't need right now, but will need later.

RAM itself is structured as a big byte array. For example, if your computer has 8 GB of RAM, that means your RAM physically provides an array of 8 billion bytes for the CPU to read and write. Every ISA includes dedicated memory read and write instructions, so code running on the CPU can store and retrieve variables in RAM.

For the same reason the CPU doesn't have many registers, the CPU also doesn't have much room to store the code it's executing; so instead, the code is staged in RAM and read (implicitly) by the CPU whenever it needs the next set of instructions. (This is the basis of the [Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture) that all modern computers are based on.)

So, in short, CPU + RAM = you have a computer. But it'd be a pretty useless computer! Practically speaking, you need a lot more 'stuff' to make a computer people can do stuff with &mdash; things like disks, SSDs, network cards, displays, keyboards, mice, touchscreens, cameras, you name it! These devices are often called [peripherals](https://en.wikipedia.org/wiki/Peripheral) because, however useful, they're not core to your computer: you can have a computer without a keyboard, but a computer without a CPU is no computer at all.

## Peripherals

How do peripherals work?

Many peripherals include some onboard circuitry that works like a rudimentary computer in its own right. This 'logic board' usually has some kind of 'controller' chip that does something similar to the job of a CPU, and logic boards often also include some RAM-like memory. More advanced peripherals might even execute some onboard code (called [firmware](https://en.wikipedia.org/wiki/Firmware)). Some of the most expensive peripherals (like high-end [DIY hackable mechanical keyboards](https://qmk.fm/keyboards/)) actually include a full-fledged onboard computer for this purpose! (Albeit, a small, slow and inexpensive one.)

The key difference between these logic boards and your CPU/RAM is programmability: logic controllers are designed to drive the peripheral device and nothing else, whereas the CPU/RAM is designed to run general-purpose code.

Putting it all together, it turns out the computer you're using to read this blog is actually a whole little network of mini-computers talking to each other. The CPU and RAM are the main programmable core of the computer, where all the software runs. This programmable core is hooked to a bunch of little auxiliary proto-computers, which each run a device such as a display, a disk, a keyboard, a touchscreen, etc. Because these are all hooked up together, code running on the CPU/RAM can interact with physical devices via their logic boards.

> Basic schematic showing your CPU+RAM when your code is running, fanning out to a 'network' which is connected to logic boards, each of which has a device icon next to it.

What we want to figure out next is how these are all hooked together. In other words, how does that 'core' computer &mdash; your CPU and RAM &mdash; exchange information with the peripheral 'auxiliary' computers and make the hardware do stuff? It looks like we need to invent [I/O](https://en.wikipedia.org/wiki/Input/output) &mdash; that is, ways for code running on a CPU to submit commands to peripherals and read back information from the device.

## I/O

Throughout the ages there have been many approaches to I/O, but the dominant approach today is a technique called [Memory Mapped I/O (MMIO)](https://en.wikipedia.org/wiki/Memory-mapped_I/O) (which, by the way, has no relation to the Unix memory mapped file I/O feature).

Here's the idea: the CPU already has instructions for reading and writing RAM, and peripherals already have some onboard memory that kind of works like RAM. So why not make it so the existing CPU read/write instructions can directly read and write the peripheral's onboard memory? (At least some of it.) That would give programs a way to submit commands to a device (memory writes) and/or receive information (memory reads), depending on what the device supports.

To make this work, we need to invent a way to make peripheral memory accessible via the CPU's 'read memory' and 'write memory' instructions. Remember that all memory, whether it's RAM or a peripheral's onboard memory, is exposed to the CPU as an array of bytes. Each index in this array is called an *address*, and the CPU's read/write instructions both take a memory address as input.

The key observation about addresses we need to make memory-mapped I/O work is that there are more possible addresses than there are bytes in RAM, so some addresses inevitably go unused. We can repurpose some of the unused addresses as a way of accessing peripheral device memory!

The first step to making this work is to shim in a new chip in between the CPU and RAM, like this:

> Diagram

For the sake of this discussion, let's call this new chip a 'memory controller.' Our hypothetical memory controller's purpose is to sort of "lie" to the CPU about memory addresses, in order to redirect some unused addresses to peripheral device memory. It's easiest to explain how this works using an example.

Say we had a very simple computer with a tiny amount of RAM (only 10,000 bytes total), and say we only have two peripherals: a keyboard and a mouse. Let's have both expose peripheral memory to the CPU using memory-mapped I/O. To make it easier to read addresses, let's say this very small computer represents memory addresses using only 16 bits, meaning all addresses from 0-65,535 are possible.

Our new memory controller chip might map all of this into the CPU's memory address space like this:

| Device   | Memory Size (in Bytes) | Mapped Addresses |
| -------- | ---------------------- | ---------------- |
| RAM      | 10,000                 | 0-10,000         |
| Keyboard | 2,000                  | 10,000-12,000    |
| Mouse    | 500                    | 12,000-12,500    |
| (unused) | ---                    | 12,500-65,536    |

Now let's say we have a program that needs to read input from the keyboard. That program will probably want to periodically read the keyboard's device memory to see which, if any keys are being pressed right now. The program code to do that knows (in ways too deep to get into here) that the keyboard's memory mapping starts at address 10,000. Let's say the program's keyboard query code wants to read from the keyboard's device memory at address 1,500. So it adds the memory address (1,500) to the keyboard's base address (10,000) to get a virtual address, 11,500. The program then executes a 'read' instruction at memory address 11,500 into a CPU register.

Our new 'memory controller' chip intercepts this read instruction and uses the table above to decode the read address. It sees 11,500 falls within the 'keyboard' mapped address range. It then subtracts the keyboard device's base address (10,000) to obtain the device-specific memory address 1,500. (This is just the opposite of the calculation our code did in the previous paragraph.) Finally, the controller issues a read from the keyboard device memory at address 1,500. The keyboard returns the memory value to the memory controller chip, and the memory controller returns the value to the CPU, which stores it in a register. The read is now complete! (Whew.)

From the programmer's perspective, all of this happens seamlessly: the programmer includes a 'read' instruction with computed address 11,500 into some CPU register, and can safely assume the value has been read into that register upon the start of the next instruction. (And, of course, on a real computer this is all handled by operating system drivers &mdash; it's uncommon to have to write this kind of code nowadays.)

> One aside about that 'memory controller' chip I keep mentioning: depending on the ISA, it has different names and appears in different places. A few years ago, if you had built your own PC or overclocked a CPU, you may have heard one of these names before.
>
> Clasically, this memory mapping functionality has been implemented as two interconnected chips that work together: a [Northbridge](https://en.wikipedia.org/wiki/Northbridge_(computing)) and a [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)). The Northbridge connects the CPU, RAM, high-speed devices and the Southbridge directly together; the Southbridge is a separate interconnects for lower-speed devices that can't keep up with RAM speeds. Wikipedia has [a nice diagram](https://en.wikipedia.org/wiki/Southbridge_(computing)#/media/File:Motherboard_diagram.svg) showing how this works.
>
> Over time, consumer hardware has been getting faster and more bandwidth-hungry, and ISA designers have responded by pushing these bridges 'up,' closer to the CPU. On today's higher-end hardware,, there often isn't a separate Northbridge chip anymore &mdash; it has been subsumed into the CPU itself to maximize transfer bandwidth.
>
> But, no matter how the ISA is designed, the core memory-mapping and redirection functionality remains critical to how computer code uses peripheral devices today.

## Copying Data

Memory-mapped I/O works pretty well for issuing commands to peripherals and moving small amounts of data. But, the approach described so far breaks down for large data transfers &mdash; let's see how.

Say you wanted to read 8 KB from a disk. (In the example earlier this would seem like a lot, but in modern terms this is a pretty small disk read.) Here's a quick schematic showing, in detail, how this would work with the memory mapping scheme described above:

> Diagram with numeric labels corresponding to the steps below

1. The CPU writes a command to the disk's memory, via a memory mapping, telling the disk what data the program would like to read.
2. Once it's ready, the disk dequeues the command and carries it out by reading from physical media (magnetic platter, solid state storage, etc). All data is temporarily stored in the disk's onboard memory.
3. The disk notifies the CPU that the read has completed, via a mechanism called an [interrupt](https://en.wikipedia.org/wiki/Interrupt) (sort of the CPU equivalent of an event callback).
4. The CPU copies data from disk memory to RAM. In a loop, it does a read from disk memory into a CPU register, then a write from the CPU register into RAM, continuing until all data has been copied.

The disk only has so much onboard memory for buffering data. If the amount of data to be read exceeds this buffer size, then the software will need to do multiple disk reads to get all the data. Each read has to follow the full process outlined above.

In modern parlance, this scheme would be called [Programmed I/O](https://en.wikipedia.org/wiki/Programmed_input–output). This scheme works, and there have even been computers that work this way, but you're unlikely to see a modern computer do this, because it's so. darn. slowww.

The problem is that loop in step 4. A CPU can only do one thing at a time, and while it's tied up copying data between the disk's onboard memory and RAM it can't run any of the code we bought our computer to run in the first place. The problem is made worse by just how long it's going to take to do that copy:

* A CPU register is only 4-8 bytes, so the loop that copies data into memory in step 4 above can easily require more than a thousand iterations, just to read from a disk one time.
* CPUs are a few hundred times faster than RAM, so a copy that takes 1,000 loops (for example) wastes hundreds of thousands of instructions worth of CPU time.
* On, and device memory isn't usually as fast as RAM, so copying might take even longer than that.

That's why modern I/O technologies employ a technique called [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA).

## DMA

The idea behind DMA is to cut the CPU out of that final copy step above by having the disk copy data instead. The CPU is free to continue doing other stuff while the disk copies data into RAM, and doesn't need to get notified until all the requested data is already in RAM, ready to go!

How does this work? Since we already have a 'memory controller' brokering access between the CPU, RAM and peripherals, this should be relatively easy to implement &mdash; all we need is a way for peripherals to execute their own memory reads and writes, without help from the CPU. In other words, instead of being a splitter that maps a bunch of devices together:

> Diagram showing a tree rooted at the CPU -> memory controller -> branching out to a bunch of individual devices

The memory controller is now a full network, allowing anyone to talk to anyone:

> Similar diagram, but now it's a bunch of bidirectional links to/from the memory controller hub, with legs pointing to the CPU, RAM, and individual devices

Here's what our disk read looks like with DMA:

> Diagram

1. The CPU writes a command to the disk's memory, via a memory mapping, telling the disk what data to read, *and* where to store the result in RAM.
  
2. Like before, the disk dequeues the command and reads from physical media, temporarily storing the result in the disk's own onboard memory.
  
3. Now for DMA: the *disk* copies data out of its onboard memory and into RAM. This is still a potentially a long loop, but now the disk executes the loop while the CPU does other stuff.
  
4. Finally, the disk notifies the CPU everything is done by sending an interrupt (event callback). There's nothing more for the CPU to do now &mdash; the requested data is already in RAM!

When DMA was first introduced, CPUs were typically much faster than peripheral devices and their corresponding memory transfers, and DMA was key to ensuring CPUs weren't being bottlenecked by peripherals. With some of today's highest-end hardware, the situation has been inverted: now we have some super fast peripherals that need DMA to avoid being bottlenecked by the CPU!

The acronym "DMA" should be vaguely familiar ... what was this blog post supposed to be about again?

## RDMA?

Remember, our original goal was to figure out what the term "RDMA" means. If you still remember what "RDMA" stands for, in light of the above, you might already see where all this is headed ...

You see, it's *technically* correct to expand "RDMA" fully to "Remote Direct Memory Access," and that's why you'll see most articles write it out that way. But it's arguably more enlightening to expand it as "Remote DMA," because the core idea of RDMA is to do a DMA transfer *across computers*.

Let's go back to our disk read example, but now with two computers: one with our code (that wants to read data into local RAM) and the other one with the disk we want to read from. A technology that supports Remote DMA will let us do a disk read as follows:

> Diagram

1. Code running on computer A sends a network command to computer B saying it would like to read data from one of computer B's disks. The command specifies which disk to read, what data should be read from that disk, and a memory address on computer A where the data should 'end up.'
  
2. Computer B receives this command and does a disk read into local memory, using regular old DMA
  
3. Now comes the RDMA part: B now copies its copy of the data out of its local memory and into A's local memory. This might require many network transfers; importantly, neither computer's CPU is involved with these transfers, and can do other stuff while the network hardware copies memory.
  
4. Finally, B sends a network response to computer A notifying it the operation is complete. When A receives this network message, it knows the data is already present in its local RAM, so there's nothing left to do

We included a disk read to show how the previous DMA example lines up with RDMA, but the disk read has nothing to do with RDMA &mdash; RDMA is just a networking technology that allows computers to copy data between local and remote RAM.

If you look at RDMA like DMA, you can kind of see what its inventors were going for. You think of conventional networks as working sort of like programmed I/O: each network transmission is small (around a kilobyte, maybe), and we need programs on each computer to coordinate on how to break down data into individual transmissions on one end and reassemble them on the other. With RDMA, this coordination still needed, but it's handled by the networking hardware alone. Like with DMA, RDMA provides two-sided benefits: the CPU is freed from the burden of managing network transfers, and the network can scale to higher speeds than the CPU could feasibly handle. RDMA networks are deployed for both reasons.

So far this all might sound a little too good to be true &mdash; if this makes both the CPU and the network more efficient, why don't all networks work this way? The short answer is that conventional networking is too hard to handle completely in hardware, so if you want Remote DMA, you first need to invent a completely different kind of network. This new kind of network requires tight coordination between peers, and can break down completely if even one peer 'misbehaves.' That kind of tight cooperation isn't really possible on the Internet, and most consumer-grade networks are primarily a link to get users connected to the Internet. So most people end up using conventional, non-RDMA networks.

The kind of tight cooperation you need for RDMA networks *is* feasible in privately operated networks built and operated by a single entity (academic institution, government agency, private business, etc). That's why you end up seeing RDMA almost ubiquitously in the context of things like supercomputers and private clouds, but pretty much nowhere else.

But for now be happy, for you know the definition of RDMA! Or do you?

## RDMA (The Colloquial Definition)

Everything we've said so far is correct in the technical sense, and if you search for RDMA on the Internet, the above is more or less what you're going to learn. But there's a second common meaning to the term "RDMA:" InfiniBand, and InfiniBand-related technologies.

InfiniBand was the first prominent technology to feature RDMA. The designers were trying to create one unified interconnect (InfiniBand) which works both *between* computers, like a conventional network, as well as *inside* a computer, to link peripherals to CPU+RAM "programmable core." In trying to do this, they found they needed to a way to extend DMA to work across computers, and so "Remote DMA" was born.

Although InfiniBand never really took off widely, it did gain a strong hold in the [High-Performance Computing](https://en.wikipedia.org/w/index.php?title=High-performance_computing&redirect=no) community, and eventually gave rise to several extensions and competing standards. This all happened organically, and the community never came up with a good umbrella term to describe all of this tech as a whole. It's also preferable to avoid saying "InfiniBand" in marketing materials, because "InfiniBand" is a trademarked term. (Please remember to include the implicit "(TM)" every time you read the word "InfiniBand" in this article.)

Without a better option, the de facto term for all these technologies ended up being "RDMA." Which means, depending on the context, "RDMA" could refer to "Remote DMA," the thing we've spent so much time describing so far, or it could mean "networking technology that features Remote DMA" (and is probably related to, or at least inspired by InfiniBand).

Not confusing at all, eh? Well, stick around for a while and you'll get used to it.

## How Does RDMA Work?

Now that we know what RDMA is, figuring out roughly how it works is a useful next step. For the remainder of this blog, we're going to explore the InfiniBand spec at a very high level. InfiniBand is a good technology to examine not only because it's fairly popular (at least in the HPC sphere), but also because it's sort of a lingua franca among RDMA technologies: knowing InfiniBand is helpful to learning other RDMA technologies the same way how knowing C it helpful in learning a variety of popular programming languages.

InfiniBand is a huge spec though &mdash; we have no hope of covering it in detail, and great detail wouldn't be very useful anyways. Instead, we're going to cover some of InfiniBand's key architectural decisions. This will be the kind of stuff you'd need to know to program distributed applications running on an InfiniBand-like RDMA network, but maybe not so much the stuff you'd need to know to actually build something like InfiniBand yourself.

## InfiniBand

When looking at a technology, it's often useful to understand the historical context in which it was invented.

InfiniBand first came about around the turn of the milliennium. At the time, the dominant specification for connecting high-bandwidth hardware peripherals to a CPU was an interconnect called [PCI](https://en.wikipedia.org/wiki/Peripheral_Component_Interconnect); however, it was becoming clear that PCI's days were numbered, as the most bandwidth-hungry top-of-the-line peripherals around were starting to run into PCI's fundamental bandwidth limitations. The industry as a whole was starting to look for a modernized, higher-bandwidth alternative.

The industry came up with several specifications to try and meet that demand. Today, we know the winner of the race was [PCI Express](https://en.wikipedia.org/wiki/PCI_Express), an interconnect that is still the dominant protocol for the highest-speed peripherals 20 years later. But there was another notable contender in this race &mdash; InfiniBand &mdash; which may never have enjoyed widespread dominance, but did find itself a comfy niche in the high-performance computing space.

InfiniBand itself was the merger of two competing standards &mdash; one called "Next Generation I/O" and another called "Future I/O" &mdash; and was heavily inspired by another specification called the [Virtual Interface Architecture](https://en.wikipedia.org/wiki/Virtual_Interface_Architecture) (VIA). The designers of all these specifications noticed the growing similarities between interconnects (which link peripherals inside a computer) and networks (which link computers), and were trying to build a single technology to serve both as a hardware interconnect and a network.

> Does it sound weird to want to unify interconnects and networks with a single technology? Why would someone do that?
>
> Certainly, the maxim "two is worse than one" applies here, and reducing CPU overhead is always a good idea. One key feature that becomes possible with a technology that bends the distinction between interconnects and networks is a feature called *disaggregation*.
>
> Imagine you're building a supercomputing cluster by networking together a bunch of identical computers. One problem you're going to run is figuring out how much disk storage to provision per machine; you waste money if you provision more disk storage and each computer needs, but provisioning too little could potentially bottleneck applications or rule some out altogether. Even if you manage to strike this balance perfectly, as the applications running on your supercomputer evolve so will the 'perfect' ratio. That makes this provisioning job seem kind of hopeless ...
>
> Wouldn't it be nice if you could build a separate sub-cluster of just disks, and have $M$ computers talking to $N$ disks? This is the core idea behind something called a [Storage Area Network](https://en.wikipedia.org/wiki/Storage_area_network) or SAN. Making this kind of thing easy yet performant is a key goal in bending the interconnect / network distinction in technologies like InfiniBand.

In the end, InfiniBand ended up resolving some of the design differences between interconnects and networks in favor of interconnects (i.e. making the network work more like an interconnect). In several key ways, the spec revolves around RDMA, and bending the design of the network itself to fit the requirements of RDMA. You could summarize these changes as follows:

1. Build a sequential network that never loses packets, to make hardware offload easier
2. Expose hardware command queues to the CPU, ordered according to network sequencing
3. Add DMA-style read/write operations built on top of this sequential, lossless network

This will all make sense in due time. For now, let's start at the bottom of the stack and look at the network itself: how does InfiniBand networking differ from more conventional networking technologies, and why was it made to work that way?

To start to answer that question, it'd be useful to remind ourselves how conventional networks work. Let's take a look at the most conventional of networking technologies today: Ethernet!

## Ethernet

> One liner for context

Ethernet networks are built by linking computers together using special-purpose computer-like networking devices called 'switches.' Each switch has a set of Ethernet ports that you can connect up to a computer, or to another switch:

> Picture

The job of an Ethernet switch is to forward transmissions, which the Ethernet specification calls 'frames.' So when an Ethernet frame is received on one port of an Ethernet switch, the switch needs to figure out which Ethernet port it should forward the frame to, and transmit a new copy of that frame out on the chosen port. Once the frame has been forwarded, the switch can discard its local copy of the frame, since its work with the frame is done.

Larger Ethernet networks can be built by linking switches together. In a multiple-switch Ethernet network, the job of each switch is to forward each incoming frame along to its next 'hop,' and a given frame may need to pass through multiple switches before reaching its destination. The sequence of switches a given frame passes through is often called a 'path' through the network.

For example, consider the following small slice of a much larger network:

> Client $A$ and server $B$ are connected through a diamond made of switches. Each possible path between the two is labeled with a different color

In this configuration, we see that when $A$ sends an Ethernet frame to $B$, there are two possible paths (red an blue) that the frame may pass through. How this path gets chosen is an interesting question, but outside the scope of today's discussion.

For various technical reasons, Ethernet networks impose fairly restrictive limits on the maximum size of an Ethernet frame (consumer-grade hardware usually limits frame sizes to 1-2 KB or so). If an application needs to transfer a buffer of data that doesn't fit in a single Ethernet frame, the software must break down the data into frame-sized chunks and transmit them all separately. Unfortunately, because of the way Ethernet works, actually getting a scheme like this to work is fraught with peril!

One problem that can arise is **out of order delivery**. Take another look at the diagram above: an Ethernet frame can only take one of the two highlighted paths through the network, but there's no reason to assume all frames transmitted from $A$ to $B$ will follow the same path! (In fact, in some network setups it may be benficial to scramble the frames along different paths on purpose.) The problem is, different paths might deliver packets at different speeds, so $B$ might need receive frames in the same order $A$ originally sent them:

> Diagram

Another problem is **congestion**. TODO

> Diagram?

TODO mention TCP.

TODO For RDMA, we absolutely don't want software to be involved in breaking up a transmission into frames and reassembling them on the other side. But solving these problems in hardware seems very challenging; even solving them in software is complex. Can we build a network that guarantees in-order delivery with no packet loss? Well, that's that idea of what InfiniBand did.

## InfiniBand Networking

> Adapt the following old draft content as an example scheme for how an InfiniBand network might work, then finish out by mentioning more advanced schemes that may be more practical in the real world, despite the higher complexity:
>
> ---
>
> There are a number of ways to implement a network that doesn't lose packets. The key observation is congestion is a problem of coordination, and we can avoid packet loss due to network congestion as long as every peer and every switch cooperates to ensure the network is never congested. This holds on networks that are owned and operated by a single entity &mdash; but this apporach doesn't work as well over the Internet, where it's not feasible (or practical) to expect so many users to cooperate.
>
> A simple scheme for lossless networking might use buffer reservations: when peer A wants to connect to peer B, both peers as well as all network switches together negotiate a single path through the network, as well as a buffer size in bytes that will be reserved along every hop of that path. As long as the sender never puts more data on the wire than can be stored in intermediate buffers, you guarantee there's always enough buffer along the connection to accept all the data you put on the wire. Nobody ever gets overloaded, so nothing ever gets dropped!
>
> It's worth noting, however, that this negotiation step may itself be expensive and require a bunch of transmissions between switches and peers. With InfiniBand, setting up and tearing down connections tends to be slow compared to conventional networking, but once you have a connection it's generally faster and higher bandwidth than conventional networking on a similar class of hardware.
>
> There are more advanced schemes that provide losslessness without explicit buffer reservations, such as adding a 'backpressure' mechanism for switch to ask everyone to stop sending it data for a while so it can drain its buffers. Links to ECN or something, if it's not too early.
>
> ---
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