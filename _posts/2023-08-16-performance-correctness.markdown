---
layout: post
title: Performance and Correctness
author: Dave
draft: true
---

Imagine you're interviewing with a company, and your interviewer tells you they hired 20 developers five years back and have since wrote a million lines of code, and all that's left to do now is write tests and get the code working .... you'd have *questions* for this person, right? And when your interviewer told you no, really, there are no unit tests, nobody has manually tested anything, no CI pipeline, no nada, it's a pile of code that we hope will one day do something useful, what would you do? Thank them politely for their time and then run away screaming, maybe?

Because, that's kind of what we do with performance!

Nobody in their right mind believes that you can build a large pile of software and *then* get it working later. By the time you circle back to making stuff work it'll be too late, the sheer volume of things that are wrong will be overwhelming, and you'll never finish. You're better off testing as you go, iteratively building your project piece by piece, never integrating something until you've tested it and you're reasonably confident it's going to work.

But when it comes to performance, we tend to do the exact opposite! It's ridiculously common for projects to focus on correctness first and explicitly leave profiling and optimization to the end. Usually they find a bit of low hanging fruit, and then get stuck; after that point, the performance numbers never meaningfully improve.

If performance is a concern for a project you're working on, then you really ought to have performance targets decided on day 1, and you ought to be writing small, repeatable "unit benchmarks" that show the latency or throughout or whatever of the thing you just built is plausible &mdash; feasibly good enough to be part of an overall solution that hits your targets. The same way you test your code and prove it's plausibly correct before integrating it into the project at large.

In other words, you should treat your performance the way you treat correctness. Because if you work on something that have performance numbers that matter, then performance *is* correctness. If something isn't fast enough yet, then it's not done yet, so don't merge it yet.

## But isn't that premature optimization?

It actually *prevents* premature optimization.

Quick story: A few years back, I was on a two-person team working on a [log-structured merge](https://en.wikipedia.org/wiki/Log-structured_merge-tree) engine. I'll spare you the gory details; for this discussion, all you need to know is that a key decision you make in a log-structured merge approach is the design you choose for the "memory table" or "memtable," which is an in-memory key-value map data structure that has to be queried on every read and updated on every write. The memtable is at high risk of becoming your bottleneck because there's only one of them, and every request needs to use it.

The traditional choice of data structure for a memtable is a binary search tree; the simple approach to making a memtable is to put a binary search tree behind a [reader-writer lock](https://en.wikipedia.org/wiki/Readers–writer_lock). But one lock for the entire memtable is too slow, isn't it? If performance is a concern, shouldn't we try to do better?

Well, in this case, we had a performance target: 1 million IOPS per server blade. We had some BST code lying around, so we stood up a little benchmark with a ton of threads reading and writing that BST through a reader-writer lock, and what do you know? We were able to push 4 million operations per second on the machines under our desks. The simplest memtable imaginable was already many times faster than we need it to be, so there was no reason to do anything fancier. Premature optimization avoided!

Having a performance target to measure against makes it a lot easier to tell when it's actually worth breaking out the fancy algorithms and optimziation tricks. For everything else, do the simplest thing possible!

## But we can always optimize it later!

Sure, you can. But the longer you wait, the harder it's going to be to zero on your bottlenecks, and you're also going to find the reworks are bigger and take more time to carry out. Most likely you'll have to give up for lack of time before you get your numbers to where you wanted them to be.

> Point 1 is profilers aren't magic. They can't tell you what your performance problems are any more than a debugger can explain what your bug is. Both are just tools you can use to check your assumptions. Both are really only useful if you have a single, well-scoped problem to investigate.
>
> Point two is 

## But how do you come up with a target?

Yeah, that's pretty hard. Sorry.

## What if we're too late?

> Start over and replay the development process *without* writing any new product code you don't need to. Pick your target numbers, start an empty project, set up a basic skeleton you can benchmark, and then start pulling in pieces of your old project chunk by chunk, benchmarking each as you go. Each time you find that you can't integrate a piece because the performance sucks, that's work you've discovered needs to be done. 

