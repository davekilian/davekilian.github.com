---
layout: post
title: RDMA
author: Dave
draft: true
---

"*Everything you wanted to know about RDMA (but were too afraid to ask)*"

If you've ended up here, you've probably been looking into RDMA, a networking technology popular for building large, high-performance private networks for things like supercomputers or private clouds. I've never been able to find a good, straightforward lowdown on RDMA, so here I'm writing one of my own.

I'm a software developer by trade, so I'll generally stick to software terminology, and explain hardware-related terms and concepts as they come up. Like many posts on this blog, one of the reasons I'm writing this is to crystallize my own understanding of the topic; so although I've tried to ensure everything below is true and correct, verify anything you read here before using it to make important decisions :-)

## What is RDMA?

As 15 seconds on Google will tell you, RDMA stands for "Remote Direct Memory Access." That's a mouthful if I've ever heard one! Let's unpack this a little.

To get started, we need to know a thing or two about your computer's hardware internals, and how your code interacts with it. Actually, quite a few things. Let's get started!

## Inside Your Computer

Here's a basic schematic of the inside of your computer.

> TODO

As we can see, your computer is a network of interconnected parts. Let's start by taking a look at the core parts of your computer: your CPU and your RAM.

The CPU is an electric circuit that implements a very basic programming language. Usually, the language and the CPU itself are co-developed; the CPU design coupled with the language design are called the CPU's "[instruction set architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture)" or ISA (as you can already see, hardware people love three-letter acronyms, i.e. TLAs). To vastly oversimplify, the heart of the CPU is a bunch of little calculator circuits &mdash; an add circuit, a multiply circuit, a divide circuit, and so on &mdash; plus a 'circuit picker.' Your program consists of a sequence of instructions, each of which is a number telling that 'picker' which circuit to pick next. By lining up these instructions strategically, you can get the CPU to compute something nontrivial. For example, here's a program (sequence of CPU instructions) which together calculate the 4h Fibonacci number:

>  TODO show a basic x86 fibonacci with each instruction annotated

As we saw in the example above, your CPU has circuitry to store a few variables you're working on. For historical reasons, these variables are called 'registers.' It's only cost-effective for a CPU to have a handful of registers (maybe a dozen or two), and programs often need to store more state than the CPU has registers. That's why your CPU is connected to RAM: RAM provides a place to store variables that aren't actively being worked on right now. As far as code is concerned, RAM is basically a big byte array: if you have 8 GB of RAM, for example, that means your RAM provides an array of 8 billion bytes to the CPU.

A computer's CPU and RAM are tightly coupled &mdash; the two are directly connected to each other, and the CPU's 'programming language' includes dedicated instructions for reading and writing elements in RAM. You need both CPU and RAM to have a working computer, because the instructions that your CPU executes are stored in RAM (the CPU pulls the next batch of instructions from RAM 'on-the-fly' as needed.). This is the basis of the [Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture), which just about every computer in existence today is based on.

So, in short, CPU + RAM = you have a computer. But it'd be a pretty useless computer!

Practically speaking, you need a lot more 'stuff' to make a computer people can do stuff with &mdash; things like disks, SSDs, network cards, graphics cards, monitors, keyboards, mice, webcams, you name it! These devices are often called [peripherals](https://en.wikipedia.org/wiki/Peripheral) because, however useful, they're not core to your computer: you can have a computer without a keyboard, but a computer without a CPU is no computer at all.

How do these peripherals work? In many cases, peripherals are controlled by onboard circuitry that works kind of like a rudimentary computer in its own right. The 'logic board' that runs a peripheral device often has a 'controller' chip that does something similar to the job of a CPU, and usually these logic boards include a small amount of memory that works like RAM. The key difference is programmability: logic controllers are designed to drive the peripheral device and nothing else; they aren't meant to run general-purpose code. Even in cases where the controller is programmable, the code that runs on it is typically purpose-built; we call it "[firmware](https://en.wikipedia.org/wiki/Firmware)" to differentiate from the kind of general-purpose software that runs on a CPU.

> Computers have come down so far in price that some high-end hardware includes a full-fledged CPU with onboard RAM in place of a custom controller chip for running the device. For these kinds of devices, the logic board isn't 'kind of like a rudimentary computer' &mdash; it's literally another computer!
>
> High-end [DIY hackable mechanical keyboards](https://qmk.fm/keyboards/) work this way. What makes these keyboards 'hackable' in the first place is the fact that the logic board is a CPU that runs code, and the code that runs the keyboard is an open source program (usually written in C).

To summarize, whatever device you're using to read this blog is actually a whole network of little computers talking to each other. Together, the CPU and RAM are the main programmable part of the computer, where all the software runs. This core computer is hooked to a bunch of auxiliary computers which each run a device (maybe a display, a disk, a keyboard, a touchscreen, etc). This ultimately allows software running on the core computer to interact with physical hardware.

But how exactly are these things all hooked together? How does that 'core' computer &mdash; your CPU and RAM &mdash; exchange information with the peripheral 'auxiliary' computers and make the hardware do stuff?

It looks like we need to invent [I/O](https://en.wikipedia.org/wiki/Input/output) &mdash; that is, ways for code running on a CPU to submit commands to peripherals and read back information from the device.

## I/O

Throughout the ages there have been several high-level approaches to I/O, but the dominant approach today is a technique called [Memory Mapped I/O (MMIO)](https://en.wikipedia.org/wiki/Memory-mapped_I/O) (which, by the way, has no relation to the Unix memory mapped file I/O feature).

Here's the idea: the CPU already has instructions for reading and writing RAM, and those peripheral devices already have some onboard memory. So why not make it so the CPU can directly read and write the peripheral's onboard memory? (At least some of it.) That would allow programs on the CPU to submit commands to the device (memory writes) and/or receive information (memory reads), as applicable for the kind of device.

To make this work, we need to invent a way to make peripheral memory accessible via the CPU's read memory and write memory instructions. Remember that all memory, whether it's RAM or a peripheral's onboard memory, is exposed to the CPU as an array of bytes. Each index in this array is called an *address*, and the CPU's read and write instructions both take a memory address as an input parameter. Originally, this address was simply interpreted as a RAM address, but now we need to make it so some addresses refer to peripherals' device memory, while other addresses continue to refer to RAM.

The key observation in making this work is that the size of RAM, however large, is finite, and there are more possible memory addresses than there are bytes in RAM. That means a bunch of possible addresses inevitably end up unused &mdash; there's simply no corresponding RAM to be addressed. We can repurpose some of the unused addresses as a way of accessing peripheral device memory.

To implement this, we start by shimming in a new chip in between the CPU and the RAM, like this:

> Diagram

For the sake of this discussion, we'll call this new chip a 'memory controller.' Our memory controller's purpose is sort of to "lie" to the CPU about memory addresses, in order to redirect some unused addresses to peripheral device memory. It's easiest to explain how this works using an example.

Say we had a very simple computer with a tiny amount of RAM (only 10,000 bytes total), and say we only have two peripherals: a keyboard and a mouse. Let's have both expose peripheral memory to the CPU using memory-mapped I/O. For readability, let's say this very small computer represents memory addresses using only 16 bits, meaning all addresses from 0-65,535 are possible.

Our new memory controller chip might map all of this into the CPU's memory address space like this:

| Device   | Memory Size (in Bytes) | Mapped Addresses |
| -------- | ---------------------- | ---------------- |
| RAM      | 10,000                 | 0-10,000         |
| Keyboard | 2,000                  | 10,000-12,000    |
| Mouse    | 500                    | 12,000-12,500    |
| (unused) | ---                    | 12,500-65,535    |

Now let's say we have a program that needs to read input from the keyboard. That program will probably want to periodically read the keyboard's device memory to see which, if any keys are being pressed right now. The program code to do that knows (in ways too deep to get into here) that the keyboard's memory mapping starts at address 10,000. Let's say the program's keyboard query code wants to read from the keyboard's device memory at address 1,500. So it adds the memory address (1,500) to the keyboard's base address (10,000) to get a virtual address, 11,500. The program then executes a 'read' instruction at memory address 11,500 into a local register.

Our new 'memory controller' chip intercepts this read instruction and uses the table above to decode the read address. It sees 11,500 falls within the 'keyboard' mapped address range. It then subtracts the keyboard device's base address (10,000) to obtain the device memory address 1,500. (This is just the opposite of the calculation our code did in the previous paragraph.) Finally, the controller issues a read from the keyboard device memory at address 1,500. The keyboard returns the memory value to the memory controller chip, and the memory controller returns the value to the CPU, which stores it in a register. The read is now complete! (Whew.)

From the programmer's perspective, all of this happens seamlessly: the programmer includes a 'read' instruction with computed address 11,500 into some CPU register, knowing that when the CPU finally moves onto to execute the next instruction, the corresponding keyboard memory will have been copied into the specified CPU register. (And, of course, on a real computer this is all handled by operating system drivers &mdash; it's uncommon to have to write this kind of code nowadays.)

> One aside about that 'memory controller' chip I keep mentioning: it has a few names, and you may even have heard one if you've ever built your own PC or overclocked a CPU. Clasically, this memory mapping functionality has been implemented as two interconnected chips that work together: a [Northbridge](https://en.wikipedia.org/wiki/Northbridge_(computing)) and a [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)). The Northbridge provides an interconnect between the CPU, RAM, high-speed devices and the Southbridge, which in turn is another interconnect for lower-speed protocols. Wikipedia has [a nice diagram](https://en.wikipedia.org/wiki/Southbridge_(computing)#/media/File:Motherboard_diagram.svg) showing how this works.
>
> Over time, consumer hardware has been getting faster and more bandwidth-hungry, and CPU designers have responded by pushing these chips 'up,' closer to the CPU. Today, there often isn't a separate Northbridge chip on higher-end hardware &mdash; it's now integrated into the CPU itself. Even so, the core memory-mapping and redirection functionality remains critical to how computer code uses peripheral devices.

## Copying Data

Memory-mapped I/O works pretty well for issuing commands to peripherals and moving small amounts of data. However, the approach as described so far breaks down for large data transfers.

Say you wanted to read 8 KB from a disk. (In the example above this would be a lot, but in modern terms this is on the small end of how much someone might try to read at a time.) Here's a quick schematic showing, in detail, how this would work:

> Diagram with numeric labels corresponding to the steps below

1. The CPU writes a command to the disk's memory, via a memory mapping, telling the disk what data the program would like to read.
   
2. Once it's ready, the disk dequeues the command and carries out by reading from physical media (magnetic platter, solid state storage, etc). All data is temporarily stored in the disk's onboard memory.
   
3. The disk notifies the CPU that the read has completed, via a mechanism called an [interrupt](https://en.wikipedia.org/wiki/Interrupt) (sort of the CPU equivalent of an event callback).
   
4. The CPU copies data from disk memory to RAM. In a loop, it executes a read from disk memory into a CPU register, then a write from the register into RAM, continuing until all data has been copied.

The size of the disk's onboard memory is limited, and if more than that much data needs to be read, then the software will need to use multiple disk reads to get all the data; each read has to follow the full process above.

In modern parlance, this scheme would be called [Programmed I/O](https://en.wikipedia.org/wiki/Programmed_input–output). This scheme works, and there have even been computers that work this way, but you're unlikely to see a modern computer do this, because it's so. darn. slowww.

The problem is that loop in step 4. While the CPU is tied up copying data between the disk's onboard memory and RAM, it's fully occupied &mdash; it can't run any of the programs we bought it to run. And it's going to take obscenely long to do that copy:

* A CPU register is only 4-8 bytes, so the loop that copies data into memory in step 4 above can easily require upwards of 1,000 iterations, just to read once from the disk.
  
* CPUs are a few hundred times faster than RAM, so a copy that takes 1,000 loops (for example) potentially means hundreds of thousands of instructions worth of lost CPU time.
  
* Oh, and device memory is typically even slower than RAM, further exacerbating the waste.

That's why modern I/O technologies employ a technique called [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA). The idea behind DMA is to cut the CPU out of the loop for that final copy step above. Given we already went through the trouble of connecting the CPU, RAM and peripherals to implement memory-mapped I/O, why not let the peripheral manage large data copies to and from RAM, instead of the CPU?

Here's what our disk read looks like with DMA:

> Diagram

1. The CPU writes a command to the disk's memory, via a memory mapping, telling the disk what data to read, *and* where to store the result in RAM.
   
2. Like before, the disk dequeues the command and reads from physical media, temporarily storing the result in the disk's own onboard memory.
   
3. Now for DMA: the disk now copies data out of its onboard memory and into RAM. This is still a potentially large number of 4-8 bytes copies, but now the *disk* does it, while the CPU does other stuff.
   
4. Finally, the disk notifies the CPU everything is done by sending an interrupt (event callback). There's nothing more for the CPU to do now &mdash; the requested data is already in memory!

When DMA was first introduced, CPUs were typically much faster than peripheral devices and their corresponding memory tranfers, and DMA was key to ensuring CPUs weren't being bottlenecked on comparatively slower hardware. Today, with some of the highest-end hardware, the situation is inverted: these super fast peripherals need DMA to avoid being bottlenecked by the CPU!

## RDMA?

"Alright," you might say, "this has all been neat and stuff, but weren't we going to talk about RDMA?" Yes &mdash; it's time to bring it all home!

If you still remember what "RDMA" stands for, you might already see where all this is headed. You see, it's *technically* correct to expand "RDMA" to "Remote Direct Memory Access," and that's why you'll see most articles write it out that way. But it's arguably more enlightening to expand it as "Remote DMA," because the core idea of RDMA is to do a DMA *across computers*.

Let's go back to our disk read example, but now with two computers: one with our code (that wants to read data into local RAM) and the other one with the disk we want to read from. A technology that supports Remote DMA will let us do a disk read as follows:

> Diagram

1. Code running on computer A sends a network command to computer B saying it would like to read data from one of computer B's disks. The command specifies which disk to read, what data should be read from that disk, and a memory address on computer A where the data should 'end up.'
  
2. Computer B receives this command and does a disk read into local memory, using regular old DMA
  
3. Now comes the RDMA part: B now copies its copy of the data out of its local memory and into A's local memory. This might require multiple network transfers. Importantly, these network transfers are managed completely by networking hardware: both computers' CPUs can do other work while the transfer is ongoing
  
4. Finally, B sends a network response to computer A notifying it the operation is complete. When A receives this network message, it knows the data is already present in its local RAM, so there's nothing else to do

We happened to have computer B do a disk read in this example, but the disk read has nothing to do with RDMA. More generally, RDMA is a networking technology that allows computers to copy between local and remote RAM; the important thing is that these transfers are managed completely by hardware, without the involvement of either computer's CPU.

If you look at RDMA like DMA, you can kind of see what its inventors were going for. If you're familiar with conventional computer networking, you can sort of see how it's like programmed I/O: each entwork transmission is small (usually around a kilobyte), and programs running on each computer must coordinate to break down the data into individual transmission on one end and reassemble them on the other. With RDMA, this coordination still needed, but it's entirely done in networking hardware, cutting the CPU out of the loop.

And, like with internal I/O, there are good reasons to want the CPU out of the loop &mdash; both to free it from the burden of handling network transfers and also to allow the network itself to scale to higher speeds than code on a CPU could feasibly handle.

So far this all sounds a little too good to be true &mdash; if this works so well, why not replace all networks to have them work this way? The short answer is that, to make it remotely possible to offload remote memory copies to networking hardware, you have to rethink how the network fundamentally works, and some of those changes are only practical in a data center setting. In particular, RDMA requires a level of cooperation you could never achieve on the scale of the Internet. These reasons are why you see RDMA almost ubiquitously in the context of big private networks, like supercomputers and private clouds, but pretty much nowhere else.

But for now be happy, for you know the definition of RDMA! Or do you?

## RDMA (The Colloquial Definition)

Everything we've said so far is correct in the technical sense, and if you search for RDMA on the Internet, the above is more or less what you're going to get. But there's a second common meaning to the term "RDMA:" InfiniBand, and the technologies it has inspired.

You see, InfiniBand was the first prominent technology to feature RDMA. The designers were trying to create one unified interconnect (InfiniBand) which works both between computers, like a conventional network, *and* inside a computer, like the interconnects that link the CPU, RAM and peripheral devices together. In trying to do this, they found they needed to a way to extend conventional DMA to work across computers, and so the term "Remote DMA" was born.

Although InfiniBand never really took off elsewhere, it gained a strong hold in the [High-Performance Computing](https://en.wikipedia.org/w/index.php?title=High-performance_computing&redirect=no) community, and eventually gave rise to several extensions and competing standards. This all happened organically, and the community never came up with a good umbrella term to describe all of them as a group. And it's preferable to avoid saying "InfiniBand" in marketing materials, because "InfiniBand" is a trademarked term. (Please remember to include the implicit "(TM)" every time you read the word "InfiniBand" in this article.)

Without a better option, the de facto term for all these technologies ended up being "RDMA." Which means, depending on the context, "RDMA" could refer to "Remote DMA," the thing we spent most of this article describing, or "networking technology that features Remote DMA" (and is probably related to, or at least inspired by InfiniBand).

Not confusing at all, eh? Well, if you stick around for a while, you'll get used to it.

## How Does RDMA Work?

We've spent an *obscene* amount of time just saying what RDMA is &mdash; although, to be fair, we may have learned some other stuff in the process. But just understanding "RDMA is hardware-offloaded cross-machine DMA" barely scratches the surface of how it works.

For the remainder of this post, we're going to explore the InfiniBand spec at a high level. InfiniBand is a good technology to examine not only because it's fairly popular and broadly deployed, but also because it's sort of a lingua franca among RDMA technologies: just like how knowing C makes it easier to learn related programming languages, so does knowing the basics of InfiniBand also help in learning other RDMA technologies.

InfiniBand is a huge spec &mdash; we have no hope of covering it in detail, and that wouldn't be so useful anyways. Instead, we're going to cover some of InfiniBand's key architectural details. This will be the kind of stuff that's useful in programming distributed applications that run over InfiniBand networks, but maybe not so much the stuff you'd need to know to actually build something like InfiniBand yourself.

Let's start with a little history...

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