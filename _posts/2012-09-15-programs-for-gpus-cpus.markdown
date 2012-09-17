---
layout: post
title: Writing Programs for GPUs and CPUs
author: Dave
draft: true
---

There are many tutorials online for 3D graphics APIs like DirectX and OpenGL.
Most start with a discussion of the standard graphics pipeline, followed by a
deep dive into function calls and sample code. But what's really going on 
under the hood? Understanding the answer is vital for writing reliable,
performant rendering code. 

## The Graphics Device

While computer graphics has its earliest roots in software-based 
rasterization, the idea of using dedicated hardware to perform 
graphics-related tasks has become ubiquitious in computing today. To 
understand why, it's important to look at what makes graphics hardware
special.

The graphics card is an add-in card you can connect to your PC's
motherboard. Despite appearances, however, graphics cards are quite complex.
These devices are miniature computers themselves, complete with 
processors (called GPUs) and onboard RAM.

A GPU can be thought of as a highly parallel CPU. Both GPUs and CPUs
perform similar tasks (they both execute programs), but they do so
in very different manners:

* A CPU consists of a handful of complex cores that run at very
  high speeds (often around 2-3 GHz).  In order to process a large data 
  set, where the same task needs to be performed on each element, the CPU 
  can process few elements at once. It moves through the data set sequentially.

  This is an example of a **MIMD** (pronounced _"mim-dee"_) architecture. MIMD
  stands for "Multiple Instruction / Multiple Data." On a MIMD architecture,
  one must execute multiple instructions in sequence to process multiple data
  elements.

* GPUs use much simpler cores running at lower speeds (.5-1 GHz). 
  They make up for this simplicity by packing many cores together (10s,
  100s, or sometimes 1000s). 

  The resulting system is **SIMD**: Single Instruction, Multiple Data.
  GPUs excel at processing data sets in which each data element can be 
  procesed on its own. 

Although a single CPU core can process a single data element faster than
a single GPU core, GPUs can eat through large datasets much faster than
CPUs due to the sheer number of cores in a GPU. GPUs are a great fit for
computer graphics, since many graphics algorithms are 
[embarassingly parallel](http://en.wikipedia.org/wiki/Embarrassingly_parallel).

For example, in many cases every pixel in an image can be procedued
independently of any other pixel. A GPU can then fill many pixels at once,
while a CPU is stuck filling them in sequentially, a handful at a time. 

## Splitting the Work

While GPUs are great for graphics tasks, their simplicity makes them 
inconvenient for general-purpose programming. Because of this, most
graphics-intensive programs split their logic between the GPU and the
CPU. 

## TODO

- CPU manages app logic, GPU is a worker. 
- CPU does more general computation, GPU handles parallel subtasks
- CPU sends commands to the GPU, giving it work items
- OpenGL is just a specification that GPUs and CPUs implement.
- It lets CPUs and GPUs talk to each other
- Ramifications: Synchronization
- Future ramifications / the age of parallelism

