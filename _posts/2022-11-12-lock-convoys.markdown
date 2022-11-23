---
layout: post
title: "A Complete Guide to Lock Convoys"
author: Dave
---

<div markdown="1" style="width: fit-content; border: 1px solid #eee; padding-left:1em; padding-right: 1em">

**Table of Contents**

1. [What's a convoy?](#part1)
2. [Why do locks convoy?](#part2)
3. [What can I do about all this?](#part3)

</div>

<div style="margin-left: 3em; margin-top: 2em"><i>The caravan carries me onward<br>On my way at last, on my way <a href="https://www.rush.com/songs/caravan/">at last</a>!</i></div>

## Picture This

You've poured hours of blood, sweat, tears and keystrokes (mostly keystrokes) into your shiny new high throughput, low latency, ultra fast *thing*. Excitedly, you fire up your favorite benchmark and watch your new code rip through requests like a hot knife through butter. For a while, your code continues along at a brisk clip.

Then all of a sudden, throughput tanks. No matter how long you wait, it never improves again.

Oh well. Disappointing, but not wholly unexpected; this is new code after all! After confirming this was no fluke, you fire up your metrics system and get to work. But what you see is weird. No resource pools are being exhausted. No data structures are getting large. Heap activity looks nominal. Background work is proceeding normally. Internal timings look fine. In fact, no one part of the code seems to be at fault.

Throughput just collapses part of the way through your benchmark, and there's no clear reason why.

Then you find something even weirder: you can sort of 'fix' the problem by stopping your benchmark workload and then starting it again. You don't have to do anything else to your server &mdash; it fixes itself as soon as you stop pushing traffic! But run a little longer and, voomp, throughput drops again. Stop and restart the benchmark? Everything is fine! Until a minute later, when throughput collapses, again. What could possibly cause something like this?

You're probably dealing with a lock convoy.

<center>
  <a name="part1"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 1: What's a convoy?
  </h1>
</center>



Lock convoys were so named in [a memo](https://jimgray.azurewebsites.net/papers/Convoy%20Phenomenon%20RJ%202516.pdf) published by IBM researchers in the late 1970s. Despite being nearly 50 years old, the paper could describe the same phenomenon in modern  computing systems almost word-for-word. Yep, convoys have been plauguing us for nigh on 50 years!

For a problem that has been causing headaches for so long, convoys seem to be little known and poorly understood, even among developers who work on the exact kinds of systems they tend to crop up in: high-performance multithreaded networked systems like databases and web servers. Why is that?

Well for one, lock convoys are a very in-the-weeds kind of problem. They arise from the interplay between the designs of your code, your CPU, your operating system kernel, and the way your locks are implemented in software. (The traffic your server is handling plays a major role too.) Most people have a murky understanding of things that go on inside CPUs and operating systems, so the details of lock convoys are murky too.

Let's do something about that!

In this post, I'm going to do a deep dive into lock convoys: what they are, how they happen, and what you can do about them. Along the way, we'll need to talk about things like CPU interrupts and operating system kernels. I'll keep it as high-level as I can, but we really do need to go into some detail to understand all of what's going on. If you want to get the most out of this post, prior familiarity with these topics would be helpful. If you want some background reading, I recommend

* *[Operating Systems: Three Easy Pieces](https://www.ostep.org)*, a wonderful (and free!) online e-textbook on operating systems and the hardware interfaces underneath.
* *[The Art of Multiprocessor Programming](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1)*, a textbook that dives deep on lock design.

Let's get started!

## A 'Request Convoy'

Let's start by looking at a situation that's very similar to a lock convoy, but doesn't involve locks at all. This will help us build intuition around how a convoy behaves, without us having to dive into locks, interrupts and kernel traps just yet.

For this thought experiment, imagine you have some kind of server that accepts requests from the Internet. This server is single-threaded: it can only process one request at a time. Any other requests that arrive concurrently must wait in a request queue.

For the next step, I'm going to need some audience participation 🙂. I need you to set up a beat of some sort. You could snap your fingers in a rhythm, or tap your foot on the floor, or whatever you want to do. Just pick a good, steady rhythm, about 1-2 beats per second.

Are you snapping, tapping, drumming along now? Good!

Each time you snap your fingers, two things happen in very quick succession:

1. **A request goes out**. The server finishes the request it was working on
2. **A request comes in**. A new request arrives on the network, and the server starts working on it

The timings are unrealistically exact, for sure, but now we have a system that satisfies two key properties:

1. **The request queue is always empty**. New requests only arrive when the server is idle
2. **The system is highly utilized**. There is almost no idle time between requests

So far, timings in this system look pretty good: the turnaround time for each request is exactly one snap, which is also as fast as the server can go. But the system is in a delicate equilibrium. Let's see what happens if we throw it off balance. Still got that rhythm going?

Imagine, on top of everything else that's going on with this system, suddenly, 16 more requests arrive all at once. All 16 new requests go straight into the request queue. How does that affect the system's timings?

Well, we still complete one request per snap, and we still receive one new request per snap: one out, one in. But now the request queue *isn't* empty to start out with. Each snap, we start with 16 requests in the queue; then one request goes out, and another comes in to replace it. The new net depth? 16 requests. That's right: the request queue depth will never change!

That is ... not good.

Before, when the request queue was always empty, there were no queuing delays in the system. The turnaround time for each request was just one snap. But now the turnaround time has ballooned to 16 snaps! For most of that time, requests are sitting latent in a queue, doing nothing. How boring.

And, it would seem there's no way for the system to recover from this situation on its own. It'd help if the client would slow down its request rate for a while, but who knows if that'll ever happen?

*And*, the effect is cumulative! What if someone dumps another 16 extra requests on us? Even if that happens hours, days, weeks later, the result will still be <u>32</u> requests in the queue, and 32-snap turnaround times &mdash; indefinitely!

(You can stop the rhythm now. Thanks for being a good sport!)

What happened here? The way a one-time blip (those extra requests arriving) caused long-term performance degradation is a telltale sign. It would seem a **convoy** has formed in the request queue.

## What's a Convoy?

In software, the word "convoy" comes from a metaphor for something similar that happens to cars driving on a highway. The short video below shows what a 'convoy' looks like in the real world:

<center>
 <iframe width="560" height="315" src="https://www.youtube.com/embed/BKQUk-9vC4s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

The "shockwave" in the video above is an example of what software people call a *convoy*. A convoy is a persistent chain reaction of waiting that can occur in any queuing system &mdash; requests in a server, cars on a highway, or threads obtaining a lock (spoiler alert).

In the video, the convoy (shockwave) is continually dissipating and growing in equilibrium: it shrinks as cars at the front of the convoy are able to accelerate out, but it grows as new cars enter the convoy at the back of the line and have to slow down. If you watch the video again, you can see how individual cars make it through the convoy, albeit slower than normal, even though the convoy itself persists. You can imagine how, in even heavier traffic, the shockwave could stick around much longer.

In the request queuing example, something very similar happens. The convoy of requests in the request queue is also dissipating and growing in equilibrium: it shrinks as the server pulls requests off the queue, but it also grows as new requests enter at the tail of the queue. Once again, requests make it through the request queue, albeit slowly, but the convoy persists so long as new requests arrive fast enough to keep feeding it. Our example was engineered in a way that allowed the convoy to persist indefinitely.

Both examples also show a key property of convoys: in the right conditions, a single mishap can form a severe, long-lasting slowdown. It only took one car braking a few seconds, or one time the request server queued too many requests, to start the chain reaction. But once a convoy has formed, it can last long after the initial triggering event is over. Exactly how long a convoy persists depends on how fast it's being cleared at the front end, relative to how fast it's being fed at the rear end.

The way that chain reaction of waiting can far outlast the original slowdown is what makes convoys difficult to investigate. By the time you notice the queue is undergoing a chain reaction of waiting, the request that messed everything up is already long gone!

## What Convoys Look Like from the Outside

We're now ready to solve the riddle posed at the beginning of this post.

If you recall, we started this post by telling a story about benchmarking a new system. We found that throughput was initially very good, but would randomly tank in the middle of a benchmarking run, and never recover. We also found that stopping and restarting the benchmarking workload caused throughput to fully recover, at least for a while.

Knowing what we know so far about convoy dynamics, do you think you could explain how all these symptoms could be the result of a convoy somewhere within the server being benchmarked? I'll give you a minute to think.

Done? Okay, here's what's likely going on:

First off, the story made it clear our request-serving system was designed to handle high rates of traffic, *and* that it's being benchmarked under high rates of traffic. We're likely pushing the limits of what the server can handle, but that shouldn't be a problem!

Initially, everything is fine: there are lots of requests, but they're all filtering through the system efficiently, just like heavy but fast-moving traffic on a highway.

Then, unbeknownst to us, there was a small mishap. Like a single car braking on a highway, or a few too many requests showing up one time in the request server example, something must have happened inside our system to delay some of our requests. Normally that kind of thing is not a problem, but this time it must have set off a chain reaction of waiting &mdash; a convoy formed, and that must be why our throughput suddenly tanked.

Then the chain reaction continued. Requests were making it through the convoy (albeit slowly), only to be replaced by new incoming requests. This kept the convoy fed, allowing it to persist more or less indefinitely &mdash; that's why throughput never recovered on its own.

But then we stopped the benchmarking traffic. All of a sudden, there were no new requests arriving, so there was nothing left to keep feeding the convoy. As the existing requests all filtered out, the convoy just dissipated, exactly like the end of the video. The convoy was thus gone by the time we restarted benchmarking traffic, which is why throughput was good again ... at least, for a while. But eventually, there was another small mishap, another convoy, and things slowed down again. Wash, rinse, repeat.

Does it all make sense now?

We've been using highways and request queues as examples because they're both familiar and easy to reason about. But convoys can appear anywhere in your system where work has to wait in a queue. Like in a lock. But boy, do convoys in locks hit so much worse than anything that happened to our request queue.

So, let's talk about locks.



<center>
  <a name="part2"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 2: Why do locks convoy?
  </h1>
</center>



## How Locks Work

Your garden-variety lock is also a queue, much like the request queues we were talking about before. But instead of queueing up requests, locks queue up threads.

There are many kinds of locks, and just about all of them can convoy in the way we describe in this post. So to keep an already long post from getting longer, we'll only consider only the simplest "mutual exclusion" or "mutex" style locks, which can be granted to only one thread at a time.

Mutex locks can be in only two states: a lock is **acquired** if some thread currently holds it, or **available** otherwise. A thread can acquire an available lock immediately, changing its state to acquired; that thread (and only that thread) can later release the lock, reverting the lock's state back to available. Doing so allows other threads to acquire the lock in the future.

What if a thread tries to obtain a lock that some other thread is already holding? This situation is called **lock contention**. A lock is said to be *contended* when multiple threads' attempts to acquire the lock overlap. Contention can refer to a single event or overall trends. When talking about trends, we say a lock is 'lightly contended' if contention is rare, or 'heavily contended' if contention arises all the time. Contention is a performance problem, and we often go to great lengths to design our systems so as to avoid contention in our locks.

Say you're designing a lock from scratch. That means you have to answer a design question: what *should* happen if threads contend for a lock? That is, what happens if a thread tries to acquire a lock that some other thread is already holding?

The vast majority of lock implementations solve this by making that thread wait. Each instance of the lock internally tracks a queue of *waiting threads*, each of which would like to acquire the lock when it is next released. When a thread tries to acquire the lock but finds it is not available, that thread adds itself to the wait queue.

What should the thread do next? There's nothing more this thread can accomplish until the lock is released, and that might not happen for a very long time. That's why most locks have the thread **yield** once it's waiting in the lock's queue. The thread calls into the operating system to essentially say "I'm all done," allowing the operating system to stop the thread and reuse the CPU core to run something else, or shut down the CPU core until there is something else to run.

Of course, this isn't the end of the story. At some point, the lock will be released, and the thread releasing the lock will have to wake the sleeping thread, to let it know it can now acquire the lock. This is done by calling into the operating system once more.

There are many ways to design the exact protocol by which a thread releasing the lock coordinates with threads waiting to acquire the lock. A simple approach, which is sketched out below, is for the thread to implicitly hand off the lock directly to one of the sleeping threads. This is nice because we don't have to mess with lock states during handoff:

```c
void acquire(my_lock *lock) {
  
  // Try to acquire without waiting
  if (acquire_lock_now(lock)) {
    return; // we got the lock!
  }
  
  // Seems someone else has the lock.
  // Notify them that we'd like to
  // obtain this lock.
  queue_node wait = (...);
  enqueue(&lock->wait_queue, &wait);
  
  // There's nothing left to do until
  // the lock is available again.
  // Yield this CPU back to the OS.
  sleep();
  
  // sleep() only returns when someone
  // wakes us up, so by convention we
  // can assume we now hold the lock.
  // So once we get here, we have the
  // lock -- we're done!
}

void release(my_lock *lock) {
  
  if (empty(&lock->wait_queue)) {
    // Nobody else wants the lock
    release_lock_now(lock);
    return;
  }
  
  // Wake up the head of the queue.
  // This notifies that thread that
  // it now owns the lock; since
  // the other thread owns the lock,
  // we've implicitly released
  // ownership of the lock. So,
  // after this, we're done.  
  queue_node *node =
      dequeue(&lock->wait_queue);
  
  wake(node->thread);
}
```

(By the way, this code is only a *sketch* of how your lock might be implemented. This code is actually horribly broken. 🙂 If you're implementing a lock yourself and you're *not* making heavy use of atomic memory instructions, your implementation is probably wrong!)

The thing is, if every lock in your program implicitly contains a queue of waiting threads, then a convoy of waiting threads can form in any lock, anywhere in your program!

Uh oh.

## Forming a Lock Convoy

Let's take a look at how in practice a convoy might appear in a lock's wait queue.

That's right, it's time for your favorite  another thought experiment! Like before, I'm going to ask you to start snapping out a steady beat. Got it going? Okay, now ...

Imagine a lock, somewhere deep inside some kind of server. Each time you snap your fingers, two things happen in very quick succession:

1. The thread currently holding the lock releases it
2. Almost immediately, another thread then comes and acquires the lock

This is (purposefully) a lot like the request queue example. Every snap, one thread goes out of the lock, another thread comes in. Like the request queue initially, this lock is highly utilized, but no thread ever has to wait in the lock's queue. That means, at least for now, there's no delay incurred by this lock. So far, so good.

This is where things get a little different from the request queue example. Imagine that, for reasons unknown, a thread holds the lock for *way* too long. In slang, we say this thread "went out to lunch" before finishing its job. Instead of releasing the lock at the end of the snap, the thread might hold the lock for two snaps, or dozens of snaps, or hundreds of snaps. A thread can go out to lunch like this for a variety of reasons; for example, if you started a file or network socket I/O while holding the lock. But this can also happen for reasons completely outside your control; we'll talk about those later. For now, what's important is the thread holds the lock way longer than normal.

While that thread is out to lunch, you keep snapping, and each snap another thread shows up to acquire the lock. Finding the lock is currently in use, all of these new threads pile up in the lock's wait queue. For the sake of discussion, let's say the lock thread was out for 16 snaps, causing 16 threads to get stuck in the lock's wait queue.

What happens next? Now the story's going to look a lot like the request queue example again, isn't it?

Sixteen snaps later, that first thread *finally* comes back and releases the lock. That allows one waiting thread to wake up and take over ownership of the lock, reducing the wait queue depth by 1. But, on the same snap, another thread will arrive and get stuck in the queue. Each snap, one thread goes out, one thread comes in, and the wait queue remains 16 threads deep. Forever. A convoy!

(Okay, you're done snapping now. Thanks for the encore! I promise you won't have to do that again for the rest of this post 🙂.)

Once again, just like the request processing example, a small mishap caused a convoy to form in our queue. However, this time it wasn't because we took on too much work &mdash;it was because a thread holding the lock was temporarily out of commission. But once that happened, the lock convoy played out the same way as the request queue: new threads kept arriving as fast as the wait queue could be cleared, causing a long-term slowdown for all threads trying to use the lock.

So far this sounds exactly like the request queue example. But it turns out, convoying in a lock is a much more serious performance problem. To understand why, we need to take a closer look at what happens to threads as they pass through the wait queue.

For that, we're going to have to learn a bit more about threads.

## A Thread's Life

Threads are a software abstraction implemented by your operating system. The principle is better explained in the [operating systems book I linked to earlier](https://ostep.org), but in brief, operating systems virtualize the CPU by time sharing: each thread gets an allotted time slice, during which it has sole control of the CPU. When that time slice is over, the operating system **preempts** the thread, which means it forcibly takes control of the CPU away from the thread. Once the thread has been preempted, the operating system can put a different thread on the CPU core, and let the new thread start a new time slice. At the end of that time slice, that thread gets preempted, and yet another thread gets scheduled. This way threads can share the CPU without having to know about, or cooperate with each other. Pretty neat!

Of course, threads don't *have* to be forcibly preempted. If a thread knows it can't accomplish anything with its remaining time, it might be polite and ask the operating system to give the CPU core away to some other thread that needs it. The process is the same as normal thread preemption, except the thread volunteers to give up core instead of the operating system forcibly wresting away control.

You may be wondering how an operating system preempts a thread. If a thread has full control of the CPU core, how do you take control back from the thread?

In all mainstream operating systems, threads are managed using a feature of CPU hardware called an **interrupt**. You can think of interrupts as the hardware equivalent of exceptions in Java or C++. In those languages, you provide code which can throw exceptions in a `try{}` block, and an exception handler in a `catch{}` block. When an exception is thrown, control jumps to the exception handler. Similarly, the operating system registers **interrupt handlers** with the CPU hardware, and when an interrupt occurs, control jumps to the operating system's interrupt handler.

Even though hardware interrupts work sort of like software exceptions, they're used for much more than just error handling. Many peripherals generate interrupts to notify the operating system that new input is available; for example, your keyboard might interrupt the CPU every time you press a key. Interrupts are also used to preempt threads: the operating system asks the CPU to set a timer in hardware, and when the timer elapses, the CPU generates an interrupt. The interrupt causes control of the CPU to jump to the operating system's timer interrupt handler, which is operating system code. From there, the operating system can then switch which thread is running on the CPU core!

What about when a thread wants to voluntarily give up its CPU core? For that it uses a **system call** (often shortened to just "syscall"). A lot of the operating system's functionality is exported to your program as system calls; for example, your code makes a system call to the operating system any time it wants to read or write a file or network socket. Thread operations that we need to implement locks, like putting a thread to sleep or waking a thread, are also exported as syscalls.

Inside, syscalls are also implemented using interrupts (or, on more modern CPUs, something very similar in spirit). To call into the operating system, your code writes a buffer to memory detailing what you want the operating system to do for you, and then your code generates an interrupt. This interrupt acts as a sort of doorbell; when you ring it, control transfers to the operating system's interrupt handler, which then finds that memory buffer and parses it to figure out what it's being asked to do.

Needless to say, there's a lot of machinery behind each and every system call, which is why it shouldn't surprise us to find system calls are pretty slow! Luckily for us, the people who designed our locks already knew this, and they had a plan.

## Avoiding Syscalls in Locks

Put yourself in the shoes of somebody designing a lock.

So far, we know that locks handle contention by putting threads to sleep while the lock is in use, and then waking them when the lock is available again. We also now know that each thread sleep or wake requires a system call, and that the overhead for a system call is pretty high. Can we avoid that overhead somehow, or are all locks doomed to be slow?

The key observation is that the thread state management is *only* needed when the lock is contended. To take advantage of this, most locks use a fast-path / slow-path design. When there is no contention for the lock, the lock can be acquired or released using the fast path, which simply changes the state of the lock in memory:  `AVAILABLE -> ACQUIRED` in `acquire()` or `ACQUIRED -> AVAILABLE` in `release()`. (This is typically done using atomic memory instructions; see the multithreading text if you want more details.)

When the lock is contended, we have to take the slow path: put the thread to sleep in `acquire()` and wake it on `release()`. That requires two system calls total: one to put the waiting thread to sleep, and one more to eventually wake it up. Not a perfect system, but at least we don't do this for *every* lock operation, right?

In fact, *real* locks use complex schemes to try and get by using the fast path even when the lock is contended. For example, the `acquire()` method might retry the fast path a few times to see if it can get lucky and acquire the lock without first going to sleep; this works as long as contention for the lock is light. Designing these retry schemes is kind of a black art though: small details about the retry policy can cause unintuitive problems at runtime.

Cool, so now we know how to design a lock. Now, here's a puzzle for you:

You know locks are usually designed using a fast path for uncontended access, and a slow path that's used when the lock is contended. How might this interplay with a lock convoy?

Here's a big hint: when we saw how a convoy forms in a lock, our lock was initially *uncontended*, right up until the convoy first formed. But after that, every access thereafter was *contended*. How does that interplay with the fast-path, slow-path design?

That's right: our lock was using the fast path for every access *before* the convoy, but after the convoy forms, we will forevermore be using the slow path!

This is what makes the lock convoy a more serious problem than what we saw with the request queue. With the request queue example, the system was still able to run at full speed; the convoy only affected request turnaround time, due to additional latency as requests waited in the queue. But with the lock convoy, the system is *actually slower than before*. CPU cycles that were being used to process requests are now being wasted doing syscalls to manage thread states inside the lock. Our system's throughput has actually dropped!

It's a vicious cycle, too: a convoying lock actually gets slower, so even more threads get stuck in the convoy, so the convoy sticks around a long time, ensnaring ever more threads.

But you know what? This isn't even the full story. And the full story is even worse 🙂

Let's talk about thread schedulers.

## How Thread Schedulers Work

Solutions in the operating systems world often come in two pieces: *mechanisms* that implement features, and *policies* that make intelligent decisions around when to use the mechanisms. We've seen the interrupt-based mechanisms for managing threads, but what are the policies? Those are implemented in an operating system component called the **thread scheduler** (also known as just "the scheduler" for short).

The major concern of any scheduler is the cost of a **context switch**. A context switch is the act of changing which thread is running on a CPU core. The main challenge of designing a scheduler comes from the fact that context switches are pretty expensive. Here's why:

Modern CPUs are complex beasts filled to the brim with complex optimizations like memory caches, long pipelines, translation lookaside buffers, branch prediction caches, instruction caches, and so on. The details of these things are beyond the scope of this guide, but all of these optimizations have two important things in common:

1. They all help the CPU run your code significantly faster
2. But they all take a while to 'warm up' before anything speeds up

That's where a context switch gets expensive. The actual code to switch threads on a CPU core doesn't have to be very expensive, but afterward, once your code starts running, it runs slower than usual: those various caches, pipelines and buffers need time to fill up. Only once that warmup completes do these optimizations kick in, allowing your CPU to resume running your code at full, optimized speed.

This means context switches come with an implicit performance penalty. Even though a context switch doesn't require very much work inside the operating system, the thread being switched to will run slowly until this warmup period has completed. This performance penalty came from the context switch and should be considered part of the cost of doing a context switch.

The people who designed your operating system's thread scheduler are aware of this. Unfortunately, there are no clever solutions this time around: our best bet is just to switch threads as infrequently as possible 🙂.

This strategy is usually implemented by tuning the system's **scheduler quantum**. The quantum is the length of the time slice the scheduler allows the thread to run, before the operating system preempts it and starts running something else. The scheduler quantum is the time that gets programmed into the CPU's hardware timer in order to generate that interrupt the operating system later uses to preempt the thread.

Setting the thread scheduler quantum tunes the system-wide cost of context switches. The longer the quantum is set, the less frequently the system switches contexts, increasing the amount of time threads run at full, warmed up speed. However, if the quantum is set too high, threads hog the CPU long enough for UIs to lag. Your computer has been configured with the largest possible quantum that doesn't seem to impact UI responsiveness. On servers that don't have a UI, the quantum might be set even higher.

This way, the quantum can be used to amortize the cost of warming up after a context switch. By running threads for as long as possible between each context switch, we maximize how long threads run with a fully warmed up CPU core. It's all a beautiful plan. You know what shoots it all to hell?

Locks.

When a lock is contended, any thread trying to acquire the lock has to put itself to sleep. That ends the thread's scheduler quantum immediately, leaving the operating system with no choice but to context switch away from that thread right away. Even if that thread's quantum hasn't elapsed. Even if that thread's quantum is *just beginning*. So much for amortization!

That adds another layer of cost overhead to a convoying lock. It's not just that we lose CPU cycles to making syscalls to park and unpark threads; now we see that convoys also force the thread scheduler to context-switch more often than it was designed to, causing the CPU to spend more time running in its suboptimal, still-warming-up state. So we're losing CPU cycles *and* the CPU is running our code slower!

And that's not all. At the risk of turning myself into a meme here, I have to tell you that this *still* isn't the complete story, and the complete story can be *even worse* 🙂.

Let's talk about fairness in locks.

## How (Not to) Design a Fair Lock

Here's a weird failure mode for a lock.

Let's say you have a lock that three threads are always contending for. Let's call those threads $A$, $B$, and $C$. The lock itself seems to be efficient: the acquire and release methods are always fast. But then you look at the acquisition order, and you notice it looks like this:

```
acquire: A
acquire: B
acquire: A
acquire: B
acquire: A
acquire: B
acquire: A
...
```

Hey wait, where did thread $C$ go? It looks like our lock doesn't protect against **starvation**.

In multithreading, starvation is when one or more threads contending for a resource never get it. Buggy locks can starve threads by never granting the lock to a thread. Buggy thread schedulers can starve threads by never scheduling the thread to a CPU core. Starvation can lead to serious problems in user applications and needs to be addressed.

To ensure a lock does not starve threads, locks are often designed to be **fair**. Fairness is sort of the opposite of starvation. In a fair lock, all threads which use a lock share it for equal time. By definition, then, a fair lock cannot starve any thread. No thread can draw the short straw because there is no short straw!

An intuitive way to create a fair lock is to serialize all threads in a first-in, first-out (FIFO) queue, similar to the wait queue in the lock we implemented earlier in this post. Each thread enters the lock at the back of the queue, and obtains the lock once it reaches the front of the queue.  A lock based on this principle is sometimes called "first-come, first-serve" or FCFS. If this is our only policy for determining which thread gets the lock next, we'd call that lock *strictly* first-come, first-serve.

This queue resolves any concern that any of the threads using the lock could starve, as each release of the lock moves a thread closer to its turn to hold the lock. That makes this design look attractive, and indeed, quite a few locks have been designed this way ... at first. Alas, as is often the case in multithreading, the intuitively correct design has subtle but very serious flaw.  In this instance, the flaw in this design is what prompted those IBM researchers to write [the original memo on convoys](https://jimgray.azurewebsites.net/papers/Convoy%20Phenomenon%20RJ%202516.pdf) in the first place.

It turns out a strict first-come, first serve lock policy makes lock convoys much more serious, changing what was a performance problem into a performance disaster! Here's why:

Let's say you have some kind of highly optimized request-processing server. You've done all the work to squeeze every ounce of performance out of your code, and your server can tear through each request in microseconds flat. Let's also say there's a lock somewhere in the request processing logic. Since all requests use the same logic, the lock will have to be acquired once per request. But as long as we don't hold the lock for very long, and there aren't too many threads, contention for this lock should be pretty low. So far, so good ... so long as the lock never convoys.

But let's say the lock *does* convoy, despite our best efforts. Now what happens?

Now that the lock is convoying, consider any thread which releases the lock. This thread pays the cost of system calling into the operating system to wake the thread which is being granted ownership of the lock. Inside that system call, the thread scheduler has a decision to make: should it keep running the current thread (the one which just released the lock), or should it context switch to the thread that just woke up (the which was just granted the lock)? The big hint is the word "context switch"  those are expensive, and schedulers try to avoid them. So, most likely, the scheduler will probably mark the woken-up thread as 'ready to run,' but won't switch to it just yet.  The thread that just released the lock gets to keep going.

What will that thread do next? Well, it was holding the lock in the middle of processing a request, so it's going to finish that up. Then it's going to go to the request queue, pick up a new request and start that one. One thing leads to another and, whoops, looks like our thread needs that lock again. Will it succeed in acquiring the lock? No! Reread the previous paragraph: the lock belongs to another thread already; one that isn't even running yet!

Looks like our favorite thread has just run out of options. It needs the lock, but the lock ain't available. In fact, the lock ain't gonna be available for a long time: in all likelihood the thread that currently owns the lock, doesn't even have a CPU core yet! So our thread is going to have to go to sleep now. That means yielding its scheduler quantum, and forcing a context switch.

But wait, this isn't a one-time deal. It happens every single time a thread tries to take that lock again. Every thread will yield its quantum once per every, single, request.

Oh dear.

It would seem, if a strictly first-come, first-serve lock convoys, then *every time* a thread wakes up and escapes the convoy, it can at best process just one request before it gets stuck in the convoy *and has to context switch again*. Here you have a thread scheduler that's trying to context switch once every few milliseconds, and threads that are forcing context switches once every few *microseconds*. Your system is drowning in context switches. Throughput isn't just going to drop; throughput is going to take a nosedive, my friend.

It turns out making a strict first-come, first-serve lock was a mistake. And it's an easy one to make &mdash;we even made it in our example lock code earlier in this post. It turns out you need to tweak lock policies if you want to avoid this worst-case convoying behavior. We'll talk more about that in the last chapter of this guide.

## The Final Piece of the Puzzle

We now have an almost complete understanding of convoys and why they're so serious. There's one last detail we need to clear up: what *exactly* causes a convoy to form in the first place? When threads are going out to lunch, figuratively, what does that mean literally?

Technically, anything that impedes a thread from making progress long enough while it's holding a lock can start a convoy. But the triggering event is almost always a context switch.

Any time you make a system call like `read()`, `write()` or `sleep()`, your thread gives up its scheduler quantum and forces the operating system to context switch. That's because the thread can't do anything else until an external condition has been met, such as a completed read, a completed write, or a thread-wake operation, respectively. That could be a very long time in the future! That's why you should never do any of these things while your code is holding a critical lock.

But not all context switches are under your control. The operating system does regularly preempt threads when their scheduler quanta elapse, and if you're unlucky, your thread can get preempted while it's holding the lock and still doing critical work. If that happens, the lock can't be released until the next time the system context switches back to your thread, and that might not happen for a very long time. Your code may not have done anything wrong, and your code might not even be able to *notice* that it went out of service for several dozen milliseconds, but the lock got blocked up all the same, and now you have a convoy on your hands.

<div style="margin-left: 3em; margin-top: 2em"><i>It is possible to commit no mistakes and still lose.<br>That is not a weakness. <a href="https://www.imdb.com/title/tt0708753/characters/nm0001772">That is life</a>.</i></div>

<center>
  <a name="part3"></a>
  <h1 style="margin-top: 3em; margin-bottom: 2em">
    Part 3: What can I do about all this?
  </h1>
</center>

We've learned a lot about operating systems, thread schedulers and locks on our way to gaining a deep understanding of where convoys come from and why they're a problem. Now comes the payoff. How do you notice you have a convoying lock, how do you *confirm* what you're dealing with really is a convoying lock, and how do you eliminate it?

To sum up the answer to all of these questions succinctly ...

![](https://i.imgur.com/0dVNyc0.jpeg)

Sorry, bud &mdash; this space is filled with gnarly problems, and you rarely get neat, cookie-cutter solutions. The symptoms are never completely clear, the problems aren't always reproducible, and your available data is never detailed enough. It's just you, what you know about the system, whatever circumstantial evidence you can cobble together, mixed with a dash of pure good luck. If it seems daunting, well, this is where you're gonna make your money as an engineer!

If you're working on the kind of system where locks can convoy, there's a decent chance this isn't news to you!

So where do we go now? Well, like I said before, a lot comes down to what you know about the system you're working on. So let's see how we can leverage all the stuff we just learned to deal with convoys. Maybe we can't eliminate them, but that doesn't mean there's nothing we can do about them. In this section of the guide, we'll explore our options.

We'll do this FAQ style.

## How do I notice I have a lock convoy?

Of all the questions answered in this part of the guide, this one has probably the most straightforward answer.

Lock convoys appear with a telltale pattern. Usually you have a system which is able to push some level of throughput initially, but later hits some kind of sudden, significant degradation in throughput, from which it never recovers. If you're in a situation where you can pause the system's workload, you'll also usually find that doing so causes the system to recover ... at least for a while.

However, a convoying lock is not the only reason your system can hit a sudden drop in throughput. It's not even close to being the most likely. Until ruled out, chances are much better that you're looking at something more banal, like

* A runaway memory leak forcing the system to page memory
* Poor heap utilization or garbage collector runs
* A depleted resource pool
* A cache that has filled or is thrashing
* An internal data structure that has grown excessively large
* Background work that is not completing on time
* Crosstalk from background work running on the same machine
* Congestion in downstream systems
* Crosstalk on the network

In the scheme of things, lock convoys are a serious, but rather exotic problem. They only start to look more likely to be a culprit when these kinds of things have been ruled out.

## How can I be *sure* it's a lock convoy?

Say you've done your homework, you've ruled out all the stuff above, but you're still seeing a throughput degradation problem you can't explain. You're starting to suspect the problem really is something more exotic, like a convoying lock. What should you do to check this hypothesis?

There are two behaviors of a convoying lock you we can leverage here. If we're lucky, we can not only confirm a lock is convoying, but also figure out which one is our culprit:

**Convoying locks make more system calls.** Use a sampling profiler, and capture traces of the system running in a 'normal' state and the 'degraded' state. Make sure the workloads running on the server are the same or comparable between the two traces. Because convoying locks have to put threads to sleep more often, and because that's done using expensive system calls, you should see a new hotspot in the 'degraded' version of the trace, where some lock is spending significant time in wait- or sleep-related system calls entered through a lock method. This not only gives you a clue toward your convoy hypothesis, but also identifies the specific lock that's impacting performance.

That said, this alone doesn't confirm you have a convoying lock. All this really shows you is there exists some lock which is normally lightly contended, but is more heavily contended when the system is degraded. It's just as possible that higher traffic in this lock is a downstream effect of whatever else caused system performance to degrade. The lock could be showing higher contention because there actually was a change in the lock's usage pattern.

As always, there are no absolutes, only educated guesses and circumstantial evidence.

**Convoying locks force more context switches.** Excessive context switches won't show up as a hotspot in your sampling profiler, because instead of wasting cycles in one part of the code, context switch overhead causes everything else to run slower. But your operating system should provide some kind of performance counter or profiler for context switches. For example, Windows provides the `CSWITCH` profiler event, which is logged every time the scheduler switches threads on a CPU core. This can be used to not only characterize how much more often server threads context-switch while degraded, but also to identify which specific lock appears to be the problem.

Neither of these approaches gives you incontrovertible evidence that the degradation problem you're investigating is indeed due to a lock convoy in a specific lock. But they're relatively simple, they're noninvasive (you don't have to modify code or recompile/redeploy), and they can help you identify any kind of lock-related bottleneck  not just ones caused by convoying.

If you absolutely must be certain that your degradation problem is truly due to a convoy in a specific lock, you will probably need to instrument both the thread scheduler and the lock to emit additional debug statistics or logs. That's not something I recommend, or can really can help you with 🙂.

## I have a convoying lock. What do I do now?

So you've done your investigation, you have a throughput degradation problem that fits the symptoms, and your profiler investigation turned up a specific lock that behaves suspiciously like it's convoying. Now what?

There are a wide variety of things you can help. Some are discussed below. As hinted earlier, though, convoys cannot be completely eliminated: if the scheduler so happens to preempt your thread while you're holding a lock, and your clients are pushing sufficiently high traffic for a chain reaction to start, you get a convoy. It's unavoidable. Our advice will instead focus on mitigation: how can we make convoys start less often and clear up sooner on their own?

Let's start with the obvious one:

### Eliminate the lock entirely

If you don't have a lock, you can't have a lock convoy. Duh! In fact, right off the bat I'm contradicting myself, by giving one way to truly eliminate lock convoys.

Unfortunately, this approach rarely pans out. The key problem is that a lock was ever needed in the first place. Unless the lock is related to a misfeature you can back out wholesale, you probably need some kind of synchronization, which means you need *something*, lock or otherwise. There's a decent chance that 'something' can also convoy, or do something similarly bad.

For example, how about replacing the lock with a lock-free algorithm? It sometimes works, but lock-free algorithms are not a panacea. Although we didn't get into atomic memory operations in this guide, it's worth mentioning that atomics are *significantly* slower than regular CPU instructions, and should thus be used minimally. An uncontended lock does pretty good at this, requiring exactly two atomics: one to acquire the lock, and one more to release. Even lock-free data structures with good big-O runtimes often still require dozens of atomics per logical operation, and many lock-free algorithms require optimistic retry loops, which can spam the system with atomics if not finely tuned. 

All this overhead should temper your enthusiasm for going lock-free. You may not be convoying, but your CPU will still be pretty unhappy with you. However, lock-free code cannot force a context switch, so this route is not completely without merit.

Barring this, what else can we do?

### Use a lock with built-in convoy protection

If your lock is naive, it might use a strict first-come, first-serve policy to grant ownership to the next waiting thread. As we discussed, that means threads will context switch on every lock operation once there's any contention for the lock. Convoys will be much more likely to form,  the performance penalty will be much higher, and it will be accordingly much harder for the convoy to dissipate on its own.

The good news is we've known about convoys for many, many years, and every mature lock package does something to address them. The [original IBM memo](https://jimgray.azurewebsites.net/papers/Convoy%20Phenomenon%20RJ%202516.pdf) first suggested granting the lock to a random sleeping thread instead of first-come, first serve order. As described in [this blog post](http://joeduffyblog.com/2006/12/14/anticonvoy-locks-in-windows-server-2003-sp1-and-windows-vista/), Windows's locks wake a sleeping thread without directly giving it ownership of the lock, instead making it race to acquire the lock synchronously when it next wakes up. That makes it possible for the releasing thread to race to reacquire the lock without first yielding its scheduler quantum. (If I'm reading it correctly, the approach Windows uses also appears near the end of the IBM memo.) While the exact approach doesn't matter, it's important for you to be using a lock written by someone who knew what a convoy was and did something about it.

If you're using a mainstream lock package, like the ones provided by the Windows, Linux or BSD kernels, you probably don't need to worry about this. But it's still worth checking your bases. Who knows, you might be surprised to find what random junk dependencies and custom implementations have managed to slip in over time...

### Use a smaller thread pool

Another effective way to reduce how often your lock convoys and how severe a performance penalty a convoy becomes, is to reduce how many threads use the lock in the first place. There are a couple of reasons this is a good idea.

The obvious reason it helps to reduce the number of threads using your lock is that convoys need to be continually fed to keep from dissipating. If there are fewer threads using the lock, then there are fewer threads to keep feeding the convoy, so it dissipates sooner on its own. Simple as that.

The subtler reason this helps is related to thread scheduling. Before, when we were talking about context switches, we mentioned a conundrum the scheduler encounters when a thread releasing the lock wakes a thread that wants to acquire the lock: should the newly woken thread be scheduled now, or should it wait until someone else's quantum expires? If the number of threads in your server $\approx$ the number of CPU cores, then this conundrum doesn't exist: any time a thread wakes up, there's almost certain to be an idle CPU core for the thread to start running on right away. This reduces the dead time during which the lock is owned by a thread which hasn't started running yet.

Unfortunately, implementing this approach in practice looks like architecturing your server around a small thread pool, with roughly the same number of threads as CPU cores on the host system, fed by a work queue of incoming requests and request continuations. This in turn requires you to implement your entire system around some kind of 'async' pattern, such as nonblocking I/O with callbacks or something. If you've already architected the system around one thread per request and blocking I/O calls, switching to an async model is likely a non-starter.

However, if you're just starting the design of your new system or you haven't gotten very far along, switching to an architecture which minimizes the number of running threads can be a very effective way to minimize lock contention throughout your program. The benefits go far beyond making convoys less frequent and less severe.

### Hold the lock less long, and use it less often

An important ingredient for forming a convoy is a steady stream of incoming requests for the lock while the lock is being held. Holding the lock less long reduces the likelihood of threads running into each other, and using the lock less often reduces the amount of overall traffic on the lock. Both can help keep the lock beneath the threshold at which a convoy can sustain itself.

In practice, this means

* Don't hold the lock for long periods of time (but you already knew that)
* Don't system call while holding the lock (no I/O, waits or heap allocations)
* Move extraneous code outside the critical section where you hold the lock

Another, often-fruitful approach is to see if you can split a single lock into $N$ (say, 4-16) locks. For example, if your lock protects a data structure on which you only do point queries, you may be able to change this:

```c
my_lock lock;
my_tree tree;
```

To something more like this:

```c
struct bucket {
    my_lock lock;
    my_tree tree;
} buckets[16];
```

Then, any time you need to get or set an item in this tree, you first hash the item and reduce that mod `sizeof(buckets) / sizeof(buckets[0])`. That gives you a single bucket; you acquire that bucket's local lock, and then query the tree like you normally would.

If splitting the lock this way is feasible, and successfully reduces the amount of traffic any one lock receives, then that reduces pressure on the lock, which in turn helps convoys dissipate faster on their own.

### Backpressure and Brownouts

If you've made it this far down the list, I have to admin I'm running out of good options for you. But there's still more to be found from scraping the bottom of the barrel.

Another approach to reducing the amount of incoming traffic on the lock, thereby helping convoys dissipate sooner on their own, is to implement a backpressure mechanism. In short, this means your server detects something is wrong, and sheds some of its load in an attempt to fix it. This is sort of like automating the process we talked about all the way back at the beginning of this guide, when we stopped and restarted the benchmarking workload and found that allowed the convoys to dissipate and the system to recover on its own.

While this can work, it's unideal for a few reasons:

**It's unclear what should trigger load-shedding, or how much load to shed.** Trying to detect lock-related problems by using timing information is fraught with peril, and locks don't generally provide real-time statistics detailing the rate of contention. So how would you know you need to slow down client requests for a while, let alone how many?

**Your clients see reduced availability.** When your system needs to shed load, it does this by sending an error back to the client that suggests the client retry later (e.g., an HTTP 503 'server busy' status code). Thus, although backpressure can be useful for protecting your server, it also can look like a brownout from your clients' perspective. The degree to which your service can and should apply backpressure to protect itself is thus linked to the SLA you provide your clients.

**You could magnify performance problems.** If your policies are not well tuned, then in the event your server hiccups, your backpressure mechanism could be too touchy and shed too much load. This kind of problem makes backpressure mechanisms dangerous: if it overreacts, it could make minor problems much more serious.

**You're treating symptoms, not the root cause.** Before you had a problem; now you have a problem and a hack to try and work around it. That means you now have two problems, and maybe a lot more.

That all said, a well-implemented online service needs to have some kind of throttling or backpressuring system for dealing with unexpected load surges or misbehaving clients, and if you have these mechanisms already, you may find it's easy enough to tune them to also help mitigate lock-related problems. Just be careful.

### Server Restarts (the cheese plan)

One pretty surefire way to resolve a convoy in a lock is to kill the server process and restart it from scratch. This might sound over the top, but if you're already running a distributed system with a seamless high availability / failover mechanism, you might be able to get away with doing this without significantly disrupting your clients. And, in fact, you might already be (or want to consider) randomly killing processes across the system anyways to continually exercise your failover code paths in production, a la [Chaos Monkey](https://en.wikipedia.org/wiki/Chaos_engineering#Chaos_Monkey).

That said, it's worth mentioning this approach still treats symptoms rather than addressing causes, which ultimately has limited effectiveness.

## The End (and Further Reading)

As the title suggested, this guide aimed to be a complete, standalone guide to understanding and addressing convoys in your locks. I'll let you be the judge of whether I was successful!

In case you're interested in some further reading on this subject, here are some more resources on lock convoys, including the ones I used years ago while first trying to make heads or tails of this space:

* [The original convoy paper from IBM](https://jimgray.azurewebsites.net/papers/Convoy%20Phenomenon%20RJ%202516.pdf)
* [acolyer's review of said paper](https://blog.acolyer.org/2019/07/01/the-convoy-phenomenon/)
* [Description of anti-convoy techniques in Windows locks](http://joeduffyblog.com/2006/12/14/anticonvoy-locks-in-windows-server-2003-sp1-and-windows-vista/)

Finally, if you enjoyed going on a romp through CPU hardware, operating system and lock internals, here are those textbook links again:

* *[Operating Systems: Three Easy Pieces](https://www.ostep.org)*, a wonderful (and free!) online e-textbook on operating systems and the hardware interfaces underneath.
* *[The Art of Multiprocessor Programming](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1)*, a textbook that dives deep on lock design.

> One final PSA: I'm on the market! If you're looking for a systems software engineer to help bust your bottlenecks like we talked about in this post, shoot me a mail! You can reach me via dave at [thisdomainname.com]
