---
layout: post
title: Performance Iteratively
author: Dave
draft: true
---

Imagine you're interviewing with a company, and your interviewer tells you they hired 20 developers five years back and have since wrote a million lines of code, and all that's left to do now is write tests and get the code working .... you'd have *questions* for this person, right? And when your interviewer told you no, really, there are no unit tests, nobody has manually tested anything, no CI pipeline, no nada, it's a pile of code that we hope will one day do something useful, what would you do? Thank them politely for their time and leave in a hurry?

Because, that's kind of what we do with performance!

Nobody in their right mind believes that you can build a large pile of software and *then* get it working later. By the time you circle back to making stuff work it'll be too late, the sheer volume of things that are wrong will be overwhelming, and you'll never finish. You're better off testing as you go, iteratively building your project piece by piece, never integrating something until you've tested it and you're reasonably confident it's going to work.

But when it comes to performance, we tend to do the exact opposite! It's ridiculously common for projects to focus on correctness first and explicitly leave profiling and optimization to the end. Usually they find a bit of low hanging fruit, and then get stuck; after that point, the performance numbers never meaningfully improve.

If performance is a concern for a project you're working on, then you really ought to have performance targets decided on day 1, and you ought to be writing small, repeatable "unit benchmarks" that show the latency or throughout or whatever of the thing you just built is plausible &mdash; feasibly good enough to be part of an overall solution that hits your targets. The same way you test your code and prove it's plausibly correct before integrating it into the project at large.

In other words, you should treat your performance the way you treat correctness. Because if you work on something that have performance numbers that matter, then performance *is* correctness. If something isn't fast enough yet, then it's not done yet, so don't merge it yet.

## But isn't that premature optimization?

It actually *prevents* premature optimization.

Quick story: A few years back, I was on a two-person team working on a [log-structured merge](https://en.wikipedia.org/wiki/Log-structured_merge-tree) engine. There are a lot of details that go into a log-structured merge approach that we don't need to talk about right now; all you need to know is that a key decision you end up having to make is the design you choose for the "memory table" or "memtable," which is an in-memory key-value map data structure that has to be queried on every read and updated on every write. The memtable is at high risk of becoming your bottleneck because there's only one of them, and every request needs to use it.

The traditional choice of data structure for a memtable is a binary search tree; the simple approach to making a memtable is to put a binary search tree behind a [reader-writer lock](https://en.wikipedia.org/wiki/Readers–writer_lock). But one lock for the entire memtable is too slow, isn't it? If performance is a concern, shouldn't we try to do better?

Well, in this case, we had a performance target: 1 million IOPS per server blade. We also had some BST code lying around, so we stood up a little benchmark with a bunch of threads reading and writing a moderately large BST through a reader-writer lock. Guess what &mdash; our little benchmark was able to push 4 million operations per second on the machines under our desks. The simplest memtable imaginable was already about four times faster than we need it to be on much lower end hardware than what we planned to deploy to. There was no reason to do anything fancier &mdash; premature optimization avoided!

Having a performance target to measure against makes it a lot easier to tell when it's actually worth breaking out the fancy algorithms and optimziation tricks. For everything else, do the simplest thing possible!

## But we can always optimize it later!

Sure, you can. But the longer you wait, the harder it's going to be to zero on your bottlenecks, and you're also going to find the reworks are bigger and take more time to carry out. Most likely you'll have to give up for lack of time before you get your numbers to where you wanted them to be.

Think about the imaginary company at the beginning of this post. What *exactly* was wrong about leaving all the testing until the end? The problem without not testing as you go is you end up having to debug all your mistakes at the same time, and in many cases you'll find in many cases that reworking the broken stuff is difficult because other code has since taken dependencies on the wrong behavior. The same problems apply when you leave performance analysis to the end of your project: you'll have to debug multiple bottlenecks at the same time, and some of the fixes will result in code churn.

Sometimes the decision to leave optimization until the end of the project is rooted in a misunderstanding of what profilers can and can't tell you. Profilers are to performance as debuggers are to correctness: walking through your code in a debugger can help you notice some assumptions you made were incorrect, but a debugger certainly can't outright explain to you what's wrong with your code and what you should do to fix it. Similarly, examining your assumptions around timing and memory usage in a profiler can help you notice some of your assumptions were wrong, but a profiler can't tell you what's wrong and how to fix it. 

Profilers are only effective when you're already dialed into a specific problem. Iterative benchmarking is how you identify those problems, and how you make sure you only have to debug one problem at a time.

## But how do you come up with a target?

Yeah, that can be kind of hard. You should still do it, though.

In some cases performance targets are trivially derived from real-world conditions. If you're making graphics for a device with a 60Hz display, then you target is 60 frames per second &mdash; simple! Other times the story is muddier; for example, you may be standing up a cloud service and wondering how many requests per second the service ought to be able to serve. If you've been working in an area long enough, you may have a gut instinct of what a good target should be. Otherwise, you'll have to figure it out for yourself.

## What if we're too late?

Say you didn't follow this advice, you got to the end of the project, and now you're stuck trying to improve your numbers. What can you do?

You can still do everything I described above, without throwing away all your code! Pick a target, stand up an empty project, set up a basic skeleton you can start benchmarking, and then start pulling in pieces of your old project one at a time. Each time you pull in some more code, get it under a relevant benchmark and make sure the performance is plausbily good &mdash; if it is, great!, move on to the next chunk. But if it's not, now you've discovered work you need to do before continuing.

The work you discover this way is the same work that you would discover if you went the opposite direction, of starting with your old project and paring down pieces that are too slow. But with this strategy, you only have to debug one performance problem at a time, because you don't merge anything until the all-up performance is in a reasonably good place. 
