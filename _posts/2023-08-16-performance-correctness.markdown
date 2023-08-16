---
layout: post
title: Performance, Iteratively
author: Dave
draft: true
---

Imagine you're interviewing with a company, and your interviewer tells you they hired 20 developers five years back and have since wrote a million lines of code, and all that's left to do now is write tests and get the code working .... you'd have *questions* for this person, right? And when your interviewer told you no, really, there are no unit tests, nobody has manually tested anything, no CI pipeline, no nada, it's a pile of code that we hope will one day do something useful, what would you do? Thank them politely for their time and leave in a hurry?

Because, that's kind of what we do with performance!

Nobody in their right mind believes that you can build a large software project and *then* get it working later. By the time you circle back to making stuff work, it'll be too late: the sheer volume of things that are wrong will be overwhelming, and you'll never finish. You're better off testing as you go, iteratively building your project piece by piece, never integrating something until you've tested it and you're reasonably confident it's going to work. At the end, your whole project ought to work, because there was never a time it didn't!

This has become common sense advice, but somehow, when it comes to performance, we still tend to do the exact opposite. It's ridiculously common for projects to explicitly leave profiling and optimization to the end. That rarely goes well. At the end of a project, you usually can find a bit of low-hanging fruit for optimization, but after that the performance numbers most likely will not improve meaningfully again. You're better off building with performance in mind throughout the project, and finishing with a project that hits your performance targets because there was never a time it didn't.

In practice, that means having performance targets decided on day 1, writing small, repeatable "unit benchmarks" for your code, and never merging anything that isn't plausibly fast enough to be part of your overall solution &mdash; the same way you test your code is plausibly correct before merging it. After all, if you work on something where performance matters, then performance *is* correctness; if something isn't fast enough yet, then it's not done yet, so don't merge it yet.

## But isn't that premature optimization?

Developing this way helps *prevent* premature optimization.

Quick story: A few years back, I was on a two-person team working on a [log-structured merge](https://en.wikipedia.org/wiki/Log-structured_merge-tree) engine. There are a lot of details that go into a log-structured merge approach that we don't need to get into right now; all you need to know is that a key decision you have to make is the design you choose for the "memory table" or "memtable," which is an in-memory map data structure that has to be queried and/or updated on every client request. The memtable is at high risk of becoming your bottleneck because there's only one, and every request uses it.

The traditional choice of data structure for a memtable is a binary search tree; the simple approach to making a memtable is to put a binary search tree behind a [reader-writer lock](https://en.wikipedia.org/wiki/Readers–writer_lock). But one lock for the entire memtable is too slow, isn't it? If performance is a concern, shouldn't we try to do better?

Well, in our case, we had a performance target: 1 million IOPS per server blade. We also had some BST code lying around, so we stood up a little benchmark with a bunch of threads reading and writing a moderately large BST through a reader-writer lock. And what do you know? Our little benchmark was able to push 4 million operations per second running on the machines under our desks. The simplest memtable imaginable was already about four times faster than we need it to be on much lower end hardware than what we planned to deploy to. There was no reason to do anything fancier &mdash; premature optimization avoided!

Having a performance target to measure against makes it a lot easier to tell when it's actually worth breaking out the fancy algorithms and optimziation tricks. For everything else, do the simplest thing possible!

## But we can always optimize it later!

Sure, you can. But the longer you wait, the harder it's going to be to zero on your bottlenecks, and you're also going to find the reworks are bigger and take more time to carry out. Most likely you'll run out of time before your numbers are where you wanted them to be.

Think about the imaginary company at the beginning of this post. What *exactly* was wrong about leaving all the testing until the end? One problem not testing as you go is you end up having to debug all your bugs at the same time; another is that fixes will require larger reworks in cases where other code took a dependency on the broken behavior. The same problems apply when you leave performance analysis to the end of your project: you'll have to debug all your bottlenecks at the same time, and some of the fixes will result in code churn.

That *we can optimize it later* attitude is often rooted in a misunderstanding about profilers. Profilers are to performance as debuggers are to correctness: walking through your code in a debugger can help you notice some assumptions you made were incorrect, but a debugger certainly can't outright explain to you what's wrong with your code and what you should do to fix it. Similarly, examining your assumptions around timing and memory usage in a profiler can help you notice some of your assumptions were wrong, but a profiler can't tell you what's wrong and how to fix it. If you were imagining you would reach the end of your project and profile iteratively until it's fast, you're liable to be disappointed with the results.

Profilers are only effective when you're already dialed into a specific problem. Iterative benchmarking is how you identify those problems, and how you make sure you're only debugging one bottleneck at a time.

## But how do you come up with a performance target before you have any code?

Yeah, that can be kind of hard. You should still do it, though.

Sometimes performance targets are trivially derived from the real world. If you're making graphics for a device with a 60Hz display, then you target is 60 frames per second &mdash; simple! Other times the story is muddier; for example, you may be standing up a cloud service and wondering how many requests per second the service ought to be able to serve. If you've been working in an area long enough, you may have a gut instinct of what a good target should be. Otherwise, you'll have to figure it out for yourself. Good luck!

If you get stuck and you don't have a domain expert to talk to, one thing you can always do is build a simulator: make a program that uses your platform in a similar way your real code eventually will. Benchmarking that on your target hardware should give you an estimate that's hopefully within an order of magnitude of what you could reasonably expect of your end product. it may not be perfect, but it's certainly better than nothing!

## What if we're too late?

Say you didn't follow this advice, you got to the end of the project, and now you're stuck trying to improve your numbers. What can you do?

You can still benchmark your project iteratively without having to throw away all your code. Just come up with a performance target, stand up an empty project, set up a basic skeleton you can benchmark, and then start pulling in pieces of your old project one at a time. Each time you pull in some more code, get it under some sort of benchmark and make sure the performance is plausbily good &mdash; if it is, great!, move on to the next chunk. But if it's not, now you've discovered work you need to do to hit your target. Finish it before pulling in the next chunk.

The work you discover this way is the same work that you would discover if you went the opposite direction, of starting with your old project and paring down pieces that are too slow. But with this strategy, you only have to debug one performance problem at a time, because you don't merge anything until the all-up performance is in a reasonably good place. 

Good luck!
