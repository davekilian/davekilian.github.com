---
layout: post
title: RDMA
author: Dave
draft: true
---

"*Everything you wanted to know about RDMA (but were too afraid to ask)*"

If you've ended up here, you're probably looking into RDMA, a networking technology popular with people building large, high-performance private networks, like for supercomputers or private clouds. I've never been able to find a good, straightforward lowdown on RDMA, so here I'm writing one of my own.

I'm a software developer by trade, so I'll generally stick to software terminology, and explain hardware-related terms as they come up. Like many posts on this blog, one of the reasons I'm writing this is to crystallize my own understanding of the topic; although I've tried to ensure everything below is true and correct, I recommend you independently verify anything you read here before using it to make important decisions :-)

## What is RDMA?

As 15 seconds on Google will tell you, RDMA stands for "Remote Direct Memory Access." That's a mouthful if I've ever heard one! Let's unpack this a little.

To get started, we need to know a thing or two about your computer's hardware internals, and how the hardware interacts with your code (software). Actually, quite a few things. Let's get started!

## Inside Your Computer

Here's a basic schematic of the inside of your computer.

> TODO

As we can see, your computer is a network of interconnected parts. Let's start by taking a look at the core parts of your computer: your CPU and your RAM.

The CPU is an electric circuit that interprets a sort of programming language that was defined alongside your CPU (together, this language and CPU design are called the CPU's "[instruction set architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture)" or ISA &mdash; as you can already see, hardware people love three-letter acronyms, i.e. TLAs). To vastly oversimplify, the CPU is basically a bunch of little calculator circuits &mdash; an add circuit, a multiply circuit, a divide circuit, and so on &mdash; plus a 'circuit picker.' Your program consists of a sequence of instructions, each of which is a number telling that 'picker' which circuit to pick next. By lining up these instructions strategically, you can get the CPU to compute something nontrivial. For example, here's a program (sequence of CPU instructions) which together calculate the 4h Fibonacci number:

>  TODO show a basic x86 fibonacci with each instruction annotated

As we saw in the example above, your CPU has circuitry to store a few variables you're working on. For arcane historical reasons, these variables are called 'registers.' It's only cost-effective for a CPU to have a handful of registers (maybe a dozen or two), and many programs need to store many more variables than the CPU has registers. That's why your CPU is connected to RAM: RAM provides a place to store variables that aren't actively being worked on right now. As far as code is concerned, RAM is basically a big byte array: if you have 8 GB of RAM, for example, that means your RAM provides an array of 8 billion bytes that your CPU can store into and retrieve.

A computer's CPU and RAM are tightly coupled &mdash; the two are directly connected to each other, and the CPU's 'programming language' includes dedicated instructions for reading and writing elements in RAM. You need both CPU and RAM to have a working computer, because the instructions that your CPU executes are stored in RAM (the CPU pulls the next batch of instructions from RAM 'on-the-fly' as needed.). This is the basis of the [Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture), which just about every computer in existence today is based on.

So, in short, CPU + RAM = you have a computer. But a pretty useless one at that.

Practically speaking, you need a lot more 'stuff' to make a computer useful &mdash; things like disks, SSDs, network cards, graphics cards, monitors, keyboards, mice, webcams, you name it! These devices are often called [peripherals](https://en.wikipedia.org/wiki/Peripheral) because, however useful, they're not core to your computer: you can have a computer without a keyboard, but a computer without a CPU is no computer at all.

How do these peripherals work? In many cases, peripherals are controlled by onboard circuitry that looks kind of like a rudimentary computer in its own right. The 'logic board' that runs a peripheral often has a 'controller' chip that does something similar to the job of a CPU, and usually these logic boards include a small amount of memory that works like RAM. The key difference is programmability: logic controllers aren't meant to run general-purpose code. Even in cases where the controller is programmable, the code that runs on it is typically called "[firmware](https://en.wikipedia.org/wiki/Firmware)" to differentiate from the kind of general-purpose software that runs on a CPU.

> Computers have come down so far in price that some high-end hardware includes a full-fledged computer in place of custom silicon for running device logic. For these kinds of devices, the 'firmware' is just a program that runs on that CPU.
>
> High-end [DIY hackable mechanical keyboards](https://qmk.fm/keyboards/) work this way. What makes these keyboards 'hackable' is the fact the firmware is just an open source C program stored in flash memory inside the keyboard itself.

To summarize, the computer you're using to read this post consists internally of a computer as well as a bunch of other computers that it likes to talk to. Somewhere inside all those computers something gets computed. (Ever notice the correlation between people who like computers and people who like recursive humor?)

I'm kidding here: it's really not so bad. The truth is, the CPU/RAM are the 'programmable' part of the computer, where all the code runs. All the other peripheral 'computers' are really just glue to allow devices like monitors, keyboards, mice and disks to plug into the main computer (which, again, is your CPU/RAM). So you basically have a core computer that runs your code, and you're hooking that to a bunch of auxiliary computers that each run a device (like displays, keyboards, mice, disks,  etc).

A reasonable next question is, what does it mean to hook all those things together? How does the main computer &mdash; that's your CPU and RAM &mdash; exchange information with peripheral devices and make them do stuff? That is, how do programs do [I/O](https://en.wikipedia.org/wiki/Input/output)?

## I/O

Throughout the ages there have been several high-level approaches to I/O, but nowadays the dominant approach is a technique called [Memory Mapped I/O (MMIO)](https://en.wikipedia.org/wiki/Memory-mapped_I/O) (which, by the way, has no relation to the Unix memory mapped file I/O feature).

Here's the idea: the CPU already has instructions for reading and writing RAM, and those peripheral devices already have some memory. So why not make it so the CPU can directly read and write the peripheral's onboard memory? That would allow programs on the CPU to submit commands (memory writes) and/or receive information (memory reads), as applicable.

To make this work, we need to find a way to make peripheral memory accessible via the CPU's read memory and write memory instructions. Remember that all memory, whether it's RAM or a peripheral's onboard memory, is exposed to the CPU as an array of bytes. Each index in this array is called an *address*, and the CPU's read and write instructions both take a memory address as an input parameter. Originally, this address was simply interpreted as a RAM address, but now we need to make it so some addresses refer to peripherals' device memory, while other addresses continue to refer to RAM.

The key observation in making this work is that the size of RAM, however large, is finite, and there are more possible memory addresses than there are bytes in RAM. That means a bunch of possible addresses inevitably end up unused, because there's no corresponding RAM to be addressed. In memory mapped I/O, we use some of those unused addresses to refer to peripheral devices' memory. Then code running on the CPU can continue to use the normal read and write instructions; depending on the address passed in as input, this causes the hardware to either serve the memory I/O from RAM or a peripheral's device memory.

To implement this, we start by shimming in a new chip in between the CPU and the RAM, like this:

> Diagram

For the sake of this discussion, we'll call this new chip a 'memory controller.' Our memory controller's purpose is sort of to "lie" to the CPU about memory addresses, in order to redirect some unused addresses to peripheral device memory. It's easiest to explain how this works using an example.

Say we had a very simple computer with a tiny amount of RAM and exactly two peripherals: a keyboard and a mouse, both of which expose peripheral memory to the CPU using memory mapping. This very small computer represents memory addresses using only 16 bits, meaning all addresses from 0-65,535 are possible.

Our new memory controller chip might map all of this into the CPU's memory address space like this:

| Device   | Memory Size (in Bytes) | Mapped Addresses |
| -------- | ---------------------- | ---------------- |
| RAM      | 10,000                 | 0-10,000         |
| Keyboard | 2,000                  | 10,000-12,000    |
| Mouse    | 500                    | 12,000-12,500    |
| (unused) | ---                    | 12,500-65,535    |

Now let's say we have a program that needs to read input from the keyboard. That program will probably want to periodically read the keyboard's device memory to see which, if any keys are being pressed right now. The program code to do that knows (in ways too deep to get into here) that the keyboard's memory mapping starts at address 10,000. Let's say the program's keyboard query code wants to read from the keyboard's device memory at address 1,500. So it adds the memory address (1,500) to the keyboard's base address (10,000) to get a virtual address, 11,500. The program then executes a 'read' instruction at memory address 11,500 into a local register.

Our new 'memory controller' chip intercepts this read and uses the table above to decode it. It sees 11,500 falls within the 'keyboard' mapped address range. It then subtracts the keyboard device's base address (10,000) to obtain the device memory address 1,500. (This is just the opposite of the calculation our code did in the previous paragraph.) Finally, the controller issues a read from the keyboard device memory at address 1,500.

From the programmer's perspective, all of this happens seamlessly: the programmer includes a 'read' instruction with computed address 11,500 into some CPU register, knowing that when the CPU finally moves onto to execute the next instruction, the corresponding keyboard memory will have been copied into the specified CPU register. (And, of course, on a real computer this is all handled by operating system drivers &mdash; it's uncommon to have to write this kind of code nowadays, unless you're working on operating systems or peripherals specifically.)

And that's memory mapped I/O, in a nutshell.

> One aside about that 'memory controller' chip I keep mentioning: it has a few names, and you may even have heard one if you've ever built your own PC or overclocked a CPU. Clasically, this memory mapping functionality has been implemented as two interconnected chips that work together: a [Northbridge](https://en.wikipedia.org/wiki/Northbridge_(computing)) and a [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)). The Northbridge provides an interconnect between the CPU, RAM, high-speed devices and the Southbridge, which in turn is another interconnect for lower-speed protocols. Wikipedia has [a nice diagram](https://en.wikipedia.org/wiki/Southbridge_(computing)#/media/File:Motherboard_diagram.svg) showing how this works.
>
> Over time, consumer hardware has been getting faster and more bandwidth-hungry, which has ended up pushing these chips 'up,' closer to the CPU. Today, there usually isn't a separate Northbridge chip &mdash; it's now integrated with the CPU itself. Even so, the core memory-mapping functionality remains critical to how computer code uses peripheral devices.

## Copying Data

Memory-mapped I/O works pretty well for issuing commands to peripherals and moving small amounts of data. However, the approach as described so far breaks down for large data transfers.

Say you wanted to read 8 KB from a disk. (In modern terms, this isn't so much to read at a time.) Here's a quick schematic showing, in detail, how this would work:

> Diagram with numeric labels corresponding to the steps below

1. The CPU does an MMIO write to a disk command buffer. The command buffer that was written tells the disk what data the program would like to read
  
2. As soon as the command has been written to the disk's command buffer, the disk picks it up and actually carries out the read from physical media (magnetic platter, solid state storage, etc). The disk parks all read data into an onboard memory buffer for now.
  
3. The disk notifies the CPU that the read has completed, via a mechanism called an [interrupt](https://en.wikipedia.org/wiki/Interrupt) (sort of the hardware equivalent of an event callback).
  
4. The CPU copies the data into RAM. In each iteration of a loop, it does an MMIO read from the disk's onboard memory buffer into a CPU register, followed by a write into RAM with the CPU register data. This continues until the entire buffer has been copied into RAM.

Note that the disk's onboard memory buffer (the one being copied into RAM at step 4) is limited in size. If the amount of data the program wants to read is larger than this buffer, then the program will need to use multiple disk reads to get all the data; each disk read has to follow the entire process above.

Today, hardware developers common call this scheme [Programmed I/O](https://en.wikipedia.org/wiki/Programmed_input–output). This scheme works, and there have even been computers that work this way, but you're unlikely to see a modern computer do this, because it's so. darn. slowww.

The problem is that loop in step 4. While the CPU is tied up copying data between the disk's onboard memory and RAM, it's fully occupied &mdash; it can't run any of the programs we bought it to run. And it's going to take obscenely long to do that copy:

* A CPU register is pretty small (4-8 bytes), so the loop that copies data into memory in step 4 above typically requires hundreds to thousands of iterations.
  
* CPUs are a few hundred times faster than RAM, so a copy that takes 1,000 loops (for example) potentially means hundreds of thousands of instructions worth of lost CPU time.
  
* Oh, and device memory is usually much slower to RAM, further exacerbating the waste

That's why modern I/O technologies employ a technique called [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA). The idea behind DMA is to cut the CPU out of the loop for that final copy step above. Given we already went through the trouble of connecting the CPU, RAM and peripherals for memory-mapped I/O, why not let the peripheral itself directly manage large data copies to and from RAM? That's exactly the point of DMA.

Here's what our disk read looks like with DMA:

> Diagram

1. The CPU does an MMIO write into a disk command buffer, just like before. Now, however, the command not only says what disk data to read, but also what memory address the CPU would like the data copied into
  
2. Like before, the disk carries out the read as soon as it appears in its command buffer. The data is read from physical media into the disk's own onboard memory
  
3. Here's the DMA part: the disk now copies the data out of its onboard memory and into RAM. This still requires a potentially large number of (4-8 byte) copies, but now the disk is doing the copy by itself &mdash; the CPU has been off doing something else since the end of step 1
  
4. Finally, once the data has been copied into memory, the disk sends the CPU an interrupt to notify it that the read completed and the requested data has been copied into the CPU-specified RAM buffer

Now the CPU is free to do other stuff while the device is handling data transfers, and as a plus we also made device access code simpler: now our code that uses peripheral devices just writes a command to the device (with MMIO) and goes on its merry way; when it gets an interrupt back from the device, the entire operation has already completed, and the CPU can directly operate on any data that was read in. Easy!

## RDMA?

That's all well and good, but wasn't this supposed to be an article about RDMA? Yes &mdash; and now it's time to bring it all home!

You see, it's *technically* correct to expand "RDMA" to "Remote Direct Memory Access," and that's why you'll see most articles write it out that way. But it's arguably more enlightening to expand it as "Remote DMA," because the core idea of RDMA is to do a DMA *across computers*.

Let's go back to our disk read example, but now with two computers: one with our code (that wants to read data into local RAM) and the other one with the disk we want to read from. A technology that supports Remote DMA will let us do a disk read as follows:

> Diagram

1. Code running on computer A sends a network command to computer B saying it would like to read data from one of computer B's disks. The command specifies which disk to read, what data should be read from that disk, and a memory address on computer A where the data should 'end up.'
  
2. Computer B receives this command and does a disk read into local memory, using regular old DMA
  
3. Now comes the RDMA part: B now copies its copy of the data out of its local memory and into A's local memory. This might require multiple individual transfers. Importantly, the transfer is managed completely by networking hardware: both computers' CPUs can do other work while the transfer is ongoing
  
4. Finally, B sends a network response to computer A notifying it the operation is complete. When A receives this network message, it knows the data is already present in its local RAM, so there's nothing else to do

We happened to have computer B do a disk read in this example, but the disk read has nothing to do with RDMA. More generally, RDMA is a networking technology that allows computers to copy between local and remote RAM; the important thing is that these transfers are managed completely by hardware, without the involvement of either computer's CPU.

If you look at RDMA this way, you can kind of see how conventional networking looks sort of liked programmed I/O: conventionally, each network transmission is small (less than a kilobyte), and programs running on each computer must coordinate to break down data copies into individual transmissions on one end and reassemble them on the other end. With RDMA, this coordination step is still needed, but it's done in networking hardware, which cuts the CPU out of the loop.

And, like with internal I/O, there are good reasons to want the CPU out of the loop, such as freeing up the CPU to go actually, you know, compute stuff. It goes the other way too: as modern networks are getting faster and faster they're running up against CPU I/O limits, and getting the CPU out of the loop often helps networks scale to higher bandwidths.

This might sound too good to be true &mdash; if this works so well, why don't all networks work this way? The short answer is that, to make it possible to offload remote memory copies to networking hardware, you have to rethink how the network fundamentally works, and some of the things you have to change really only work and/or are cost-effective in a data center setting. That's why you see RDMA in the context of supercomputers and private clouds, but rarely elsewhere.

Which brings us to an important addendum: there's another definition of RDMA ...

## RDMA (The Colloquial Definition)

Everything we've said so far is correct in the technical sense, and if you search for definitions of RDMA on the Internet, the above is more or less what you're going to get. But there's a second common meaning to the term "RDMA:" InfiniBand, and technologies inspired by InfiniBand.

InfiniBand was the first prominent technology to feature RDMA. The designers were trying to create one unified interconnect (InfiniBand) which works both between computers, like a conventional network, as well as inside a computer, like the interconnects that link the CPU, RAM and peripheral devices together. In trying to do this, they found they needed to a way to extend DMA to work across computers, and so the term "RDMA" was born.

Although InfiniBand never really took off broadly, it gained a strong hold in the [High-Performance Computing](https://en.wikipedia.org/w/index.php?title=High-performance_computing&redirect=no) community, and eventually gave rise to several extensions and competing standards. This all happened organically, and we as a community never came up with a good umbrella term for all this tech. We also wanted to avoid saying InfiniBand in marketing materials, because InfiniBand is a trademarked term.

Hence the de facto term for all these technologies ended up being "RDMA." Which means, depending on the context, "RDMA" could refer to "Remote DMA," the thing we spent most of this article describing, or "networking technology that features Remote DMA" (and is probably related to, or at least inspired by InfiniBand).

Not confusing at all, eh? Well, if you stick around for a while, you'll get used to it.

## InfiniBand

In this post we introduced RDMA and showed how it relates to venerable I/O mechanisms used across pretty much all modern computers. However, there's a lot more technical depth to the topic, and we've barely scratched the surface: things like how InfiniBand works, and what changes it makes to conventional inter-computer networks to make hardware-offloaded RDMA feasible.

All of this and more is covered in [a second post on InfiniBand](/ib-overview.html)

