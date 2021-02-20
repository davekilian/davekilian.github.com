---
layout: post
title: RDMA
author: Dave
draft: true
---

"*Everything you wanted to know about RDMA (but were too afraid to ask)*"

If you've ended up here, you may be looking into RDMA, a networking technology popular with people building large, high-performance private networks, like for supercomputers or private clouds. I've never been able to find a good, straightforward lowdown on RDMA, so here's one of my own. As a software developer, I'll try to stick to software terminology, and explain hardware-related terms as they come up.

Like many posts on this blog, one of the reasons I'm writing this is to crystallize my own understanding of the topic. While I've tried to ensure everything below is true and correct, don't use any of the information you read here to make major decisions without verifying it yourself first :-)

## What is RDMA?

As just about any top hit on Google will tell you, RDMA stands for "Remote Direct Memory Access." That's a mouthful if I've ever heard one! Let's unpack this a little.

To get started, we need to know a thing or two about your computer's hardware internals, and how the hardware interacts with your code (software).

## Inside Your Computer

Here's a basic schematic of the inside of your computer.

> TODO

As we can see, your computer is a network of interconnected parts.

The core parts of your computer are your CPU and your RAM. The CPU is an electric circuit that interprets a sort of obtuse programming language, often referred to as its "instruction set architecture" or ISA. (As you can see already, hardware people love three-letter acronyms, i.e. TLAs). To vastly oversimplify, the CPU is basically a bunch of little calculator circuits &mdash; an add circuit, a multiply circuit, a divide circuit, and so on &mdash; and an instruction is just a number which the CPU uses to decide which circuit to run the data through 'next.' A program is just a sequence of instructions that do something together. For example, here's a program (sequence of instructions) that calculate the 4th Fibonacci number:

>  TODO show a basic x86 fibonacci with each instruction annotated

Your CPU has a little onboard circuitry for storing the variables you're working on &mdash; for somewhat arcane reasons, these are called "registers" &mdash; but most programs need to store a lot more temporary data than the CPU can feasibly build onboard. That's why your CPU is connected to another device, called RAM. As far as your code is concerned, RAM is basically a big old byte array; if you have 16 GB of RAM, for example, that means your RAM is physically an array of 16 billion bytes that your CPU can store into and retrieve.

A computer's CPU and RAM are tightly coupled &mdash; the two are physically connected to each other, and the CPU's instruction set (ISA) includes special instructions for reading and writing bytes in RAM. You need both CPU and RAM to have a working computer, because the instructions that your CPU executes are stored in RAM and are retrieved 'on-the-fly' as the CPU needs them. This is the basis of the [Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture) on which just about every computer in existence today is based on.

So, in short, CPU + RAM = you have a computer.

But not a very useful one! You need a lot more stuff to make your computer useful. What about storage devices, like disks and SSDs? Network cards? Graphics and monitors? Your keyboard and mouse? These devices are called [peripherals](https://en.wikipedia.org/wiki/Peripheral) because, however useful, they're not core to your computer &mdash; you can have a computer without a keyboard, but a computer without a CPU is no computer at all!

Peripherals are sort of computers in their own right. Each usually has a little chip called a "logic board" which consists of, among other things, a 'controller' that does a job similar to what the CPU does for a full computer, and a little bit of onboard memory for storing commands and other temporary data. Some newer devices, like fancy DIY hackable keyboards, literally use low-cost CPUs in place of a purpose-specific logic controller!

## I/O

So, how do we hook everything together? How does the main computer &mdash; that's your CPU and RAM &mdash; exchange information with peripheral devices and make them do stuff?

The act of code sending a command to a peripheral or reading back the results is called [I/O](https://en.wikipedia.org/wiki/Input/output). Throughout the ages there have been many basic approaches to I/O, but nowadays the dominant approach is a technique called [Memory Mapped I/O (MMIO)](https://en.wikipedia.org/wiki/Memory-mapped_I/O) (which, by the way, has no relation to the Unix memory mapped file I/O feature).

Remember that the CPU already has first-class instructions for reading and writing RAM, and that peripheral devices have some onboard memory for buffering incoming commands from the CPU? Well here's the idea behind MMIO: let's make it peripheral device memory directly readable and writable by the CPU.

Remember from before that RAM is just a big byte array. Each index in this array is called an 'address.' Usually when a program needs to read or write RAM, you run a 'copy' instruction on the CPU that either reads from RAM starting at a given memory address into a CPU register (onboard variable), or writes from a CPU register to RAM starting at a given memory address. The specifics of the copy instructions are implemented directly in hardware.

The idea behind memory mapped I/O is to use the same 'copy' instruction to access not only RAM, but also peripheral device memory. To make this work, we shim in a new chip, which you might call a 'memory controller,' in between the CPU and the RAM, like this:

> Diagram

The purpose of this new chip is more or less to 'lie' to the CPU about memory addresses, by mapping RAM and each peripheral's device memory into a single 'address space' that the CPU uses. This is easier to explain using an example.

Say we had a very simple computer with some RAM and exactly two peripherals: a keyboard and a mouse. Our new memory chip might map the RAM and peripheral device memory into the CPU's address space like this:

| Device   | Memory Size (in Bytes) | Mapped Addresses |
| -------- | ---------------------- | ---------------- |
| RAM      | 10,000                 | 0-10,000         |
| Keyboard | 2,000                  | 10,000-12,000    |
| Mouse    | 500                    | 12,000-12,500    |

Let's say we have a program that needs to read input from the keyboard. That code might periodically read the keyboard's device memory to see which, if any keys are being pressed right now. This code knows (in ways too deep to get into here) that the keyboard's memory mapping starts at address 10,000, and (let's say) it wants to read the keyboard's device memory at offset 1,500. So it adds the memory offset (1,500) to the keyboard's base address (10,000) to get 11,500. The program then includes a memory read instruction at address 11,500 to go read from the keyboard device memory.

Our new 'memory controller' chip intercepts this read and uses the table above to decode it. It sees 11,500 falls within the 'keyboard' mapped address range. It then subtracts the keyboard device's mapping address (10,000) to obtain the read offset 1,500. (We just did the opposite calculation that the program did in the previous paragraph.) Finally, the controller does a read from the keyboard device memory at address 1,500. 

From the program's perspective, all of this happens seamlessly: the program includes a 'read' instruction with computed address 11,500 into some onboard CPU register, and when the next instruction executes, the corresponding keyboard memory will have been copied into that CPU register.

And that's memory mapped I/O, in a nutshell.

> One aside about that 'memory controller' chip I keep mentioning: it has a name, and you may even have heard of it (if you've researched and built a PC before, for example). Clasically, it has been implemented as two interconnected chips that work together: a [Northbridge](https://en.wikipedia.org/wiki/Northbridge_(computing)) and a [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)). The Northbridge connects the CPU, RAM, high-speed devices and the Southbridge together. The Southbridge fans out to a bunch of lower-speed protocols. Wikipedia has [a nice diagram](https://en.wikipedia.org/wiki/Southbridge_(computing)) showing how this works.
>
> Over time, this functionality has been moving up into the CPU as consumer hardware has gotten faster and more bandwidth-hungry. Today, the Northbridge is usually a part of the CPU itself, rather than its own chip on the motherboard; however, its core function remains the same: arbitrate access between the CPU, RAM and peripherals.

## Copying Data

Memory mapped I/O works pretty well for issuing commands to peripherals and moving small amounts of data. However, the approach as described so far breaks down for large data transfers.

Say you wanted to read 8 KB from a disk. Here's a quick schematic showing, in detail, how this would work:

> Diagram with numeric labels corresponding to the steps below

1. The CPU does an MMIO write to a disk command buffer. The command buffer that was written tells the disk what data the program would like to read
2. As soon as the command has been written to the disk's command buffer, the disk picks it up and actually carries out the read from physical media (magnetic platter, solid state storage, etc). The disk parks all read data into an onboard memory buffer
3. The disk notifies the CPU that the read has completed, via a mechanism called an [interrupt](https://en.wikipedia.org/wiki/Interrupt) (sort of the hardware equivalent of an event callback)
4. The CPU copies the data into RAM. In a loop, it does an MMIO read from the disk's onboard memory buffer into a CPU register. Then it does a write into RAM with the CPU register data. Once that's done, it can turn around and read a few more bytes of data from the disk's memory

Note that the disk's onboard memory buffer (the one being copied into RAM at step 4) is limited in size. If the amount of data the program wants to read is larger than this buffer, then the program will need to use multiple disk reads to get all the data; each disk read follows the entire process above.

This scheme works, and there have been computers that work this way, but you're unlikely to see a modern computer do this, because it's so. darn. slowww.

The problem is that loop in step 4. While the CPU is tied up copying data between the disk's onboard memory and RAM, it can't run programs or in general do any of the useful stuff we want it to do. And it's going to take obscenely long to do that copy: a CPU register is pretty small (4-8 bytes), so the loop that copies data into memory in step 4 above typically requires hundreds to thousands of iterations. CPUs are a few hundred times faster than RAM, so a copy that takes 1,000 loops (for example) potentially means hundreds of thousands of instructions worth of lost CPU time. And, it's actually worse than that, because the disk's device memory is probably itself tens to hundreds of times slower than RAM ...

That's why every modern I/O technology includes a technique called [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA). The idea behind DMA is to cut the CPU out of the loop for that final copy step above. The CPU, RAM, and device are already interconnected anyways in order to support memory-mapped I/O, so why not let the device itself manage large data copies between the device's local memory and RAM?

With DMA, the disk read now looks like this:

1. The CPU does an MMIO write into a disk command buffer, just like before. Now, however, the command not only says what disk data to read, but also what memory address the CPU would like the data copied into
2. Like before, the disk carries out the read as soon as it appears in its command buffer. The data is read from physical media into the disk's own onboard memory
3. Here's the DMA part: the disk now copies the data out of its onboard memory and into RAM. This still requires a potentially large number of (4-8 byte) copies, but now the disk is doing the copy by itself &mdash; the CPU has been off doing something else since the end of step 1
4. Finally, once the data has been copied into memory, the disk sends the CPU an interrupt to notify it that the read completed and the requested data has been copied into the CPU-specified RAM buffer

Now we're able to use the CPU much more efficiently while I/O is in progress: from the software's point of view, all we need to do is tell the disk to read into a given RAM address, and once we get the interrupt (event callback) that the read is done, the data is already in the buffer and ready to go. Hooray!

## RDMA?

Wait, this isn't an article about I/O techniques ... what does any of this have to do with RDMA?

Well, you see, it's *technically* correct to expand "RDMA" to "Remote Direct Memory Access," and you'll see exactly that if you search the web for "RDMA." But, it's arguably more enlightening to expand it as "Remote DMA," because the core idea of RDMA is to do a DMA I/O *across computers*.

Let's go back to our disk read example, but now with two computers: one with our code (that wants to read data into local RAM) and the other one with the disk we want to read from. A technology that supports Remote DMA will let us do a disk read as follows:

> Diagram

1. Code running on computer A sends a network command to computer B saying it would like to read data from one of computer B's disks. The command specifies which disk to read, what data should be read from that disk, and a memory address on computer A where the data should 'end up.'
2. Computer B receives this command and does a disk read into local memory, using regular old DMA
3. Now comes the RDMA part: B now copies its copy of the data out of its local memory and into A's local memory. This might require multiple individual transfers. Importantly, the transfer is managed completely by networking hardware: both computers' CPUs can do other work while the transfer is ongoing
4. Finally, B sends a network response to computer A notifying it the operation is complete. When A receives this network message, it knows the data is already present in its local RAM, so there's nothing else to do

We happened to have computer B do a disk read in this example, but the disk read has nothing to do with RDMA. More generally, RDMA is a networking technology that allows computers to copy between local and remote RAM; the important thing is that these transfers are managed completely by hardware, so that neither computer's CPU gets involved.

Why does this matter? Conventional networking technologies look more like the Programmed I/O approach we described earlier, where the CPU got tied up on copying data. In conventional networks, each network transmission is small (less than a kilobyte), and programs running on each computer must be coordinate to  break down data copies into individual transmissions on one end and reassemble them on the other end. With RDMA, this coordination step is still needed, but it's done in networking hardware, which cuts the CPU out of the loop.

There are two good reasons to want the CPU out of the loop. First, as mentioned earlier, the CPU cycles that are being spent on networking overhead might be better spent, you know, computing stuff.

Secondly, modern networks are getting faster and faster, and once the network starts running fast enough, needing to pass everything through the CPU limits how fast the network can transfer data. In other words, the CPU becomes a bottleneck! Cutting the CPU out has allowed the networking hardware to continue to scale past CPU I/O bottlenecks.

This all might sound too good to be true &mdash; if this works so well, why don't all networks work this way? The short answer is that, to make it possible to offload remote memory copies to networking hardware, you have to rethink how the network fundamentally works, and some of the things you have to change really only work and/or are cost-effective in a data center setting. That's why you mostly see RDMA in the context of supercomputers and private clouds.

Which brings us to an important addendum: there's another definition of RDMA ...

## RDMA (The Colloquial Definition)

Everything we've said so far is correct in the technical sense, and if you search for definitions of RDMA on the Internet, the above is more or less what you're going to get. But there's a second common meaning to the term "RDMA:" InfiniBand, and technologies inspired by InfiniBand.

InfiniBand was the first prominent technology to feature RDMA. The designers were trying to create one unified interconnect (InfiniBand) which works both between computers, like a conventional network, as well as inside a computer, like the interconnects that link the CPU, RAM and peripheral devices together. In trying to do this, they found they needed to a way to extend DMA to work across computers, and so the term "RDMA" was born.

Although it never really took off elsewhere, it gained a strong hold in the [High-Performance Computing](https://en.wikipedia.org/w/index.php?title=High-performance_computing&redirect=no) community, and eventually gave rise to several extensions and competing technologies. This all happened organically, and we never came up with a good umbrella term for InfiniBand and all the other technologies that were spun off from or inspired by InfiniBand. Plus, InfiniBand itself is a trademarked term.

Because of all this, the de facto term for all these technologies ended up being "RDMA." So, depending on the context, "RDMA" could refer to "Remote DMA," the thing we spent most of this article describing, or it coudl mean "networking technology that features Remote DMA" (and is probably related to, or at least inspired by InfiniBand).

Not confusing at all! Well, if you stick around for a while, you'll get used to it.

## InfiniBand

In this post we introduced RDMA and showed how was inspired by verable I/O mechanisms used across pretty much all modern computers. However, there's a lot more technical depth to the topic, and we've barely scratched the surface: things like how InfiniBand works, and what changes it makes to conventional inter-computer networks to make hardware-offloaded RDMA feasible.

All of this is covered in a second post: [InfiniBand](/ib-overview.html)