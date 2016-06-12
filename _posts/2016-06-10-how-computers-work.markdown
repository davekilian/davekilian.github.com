---
layout: post
title: How Computers Work
author: Dave
draft: true
---

Brain dump

The typical way to write a how computers work article is to break a computer down into its hardware parts and explain them. When that article gets to the CPU though, the explanation tends to glaze over and say the CPU is the 'chip that come it's things.' That's like saying the engine is the thing that moves the car. The next natural question is: how does that work?

So basically you explain Von Neumann architecture from first principles. You start with the idea of a circuit which produces a result. Theb another circuit that prod Es a different result. Then you build up a master circuit that decides a 'which circuit' value and from there chooses which sub circuit to use. Now you have an instruction. Now introduce memory and load the instruction from memory. Now build another circuit around the control circuit so you can progressively load instructions from memory and execute each one.

Congrats: you have a program!

Now do this 2 billion times a second and you have a modern CPU.

This is called the Von Neumann architecture.

The job of a programmer is to take a complex task and break it down into a sequence of instructions that a computer can execute to produce the result. Long ago, Grace Hopper had the neat idea of writing a compiler: instead of programmers putting instructions together, we can write a program to generate instructions for a more human-readable representation. Now you have modern programming languages! Huzzah

