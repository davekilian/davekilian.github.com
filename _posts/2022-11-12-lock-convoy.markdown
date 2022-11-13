---
layout: post
title: "The Complete Guide to Lock Convoys"
author: Dave
draft: true
---



> Outline
>
> Hook: You're benchmarking your XYZ and suddenly throughput just tanks. You stop and restart the benchmark, and now it's fine again ... until it tanks. Huh?
>
> Introduction. Make it clear up front there's some confusion from different people using the term "lock convoy" to mean several different but related things.
>
> Thought experiment with a single-threaded system that can do one request at a time. Snap your fingers in a rhythm. On each snap, imagine the current request completes just as the next request arrives. You have a system with zero contention, but very high utilization.
>
> Now imagine what happens if someone goes and dumps 16 more requests into your system, out-of-band. The 'one request per snap' workload is not disrupted. Now what happens? Your queue depth will be 16 ... forever. Your system latencies just went up 16x!
>
> It gets worse. As long as your original workload keeps up, this situation is self-sustaining. Each snap, you complete the request at the front of the queue, but another request appears at the end of the queue to compensate. It's going to be 16x latency, just from queuing, forever.
>
> It gets worse. If someone comes out of band one more time, and adds 16 more requests, now your queue depth is 32. Forever. The effect is cumulative.
>
> Your request queue has formed a "convoy." The term *convoy* doesn't have a precise definition. You can think of a convoy as a deep queue that causes poor performance, manifested as high latency, low throughput, or both.
>
> We've been talking about request queues because they're an obvious place work in your system gets queued. Request queues are easy to reason about. But you can get convoys anywhere your system queues up work that can be done right away.
>
> Let's talk about locks.
>
> Your garden-variety lock is also, implicitly, a queue much like the one we described above. When a thread needs to acquire a lock that someone else is holding, you have to do something with the thread that's trying to acquire the lock. Most reasonable schemes involve waiting &mdash; adding the thread to a queue of threads that would like to acquire the lock, and then putting the thread to sleep to yield the processor to other work. That queue is a wait queue, and if many threads show up to wait in the lock, you can get a deep queue and long wait times &mdash; another convoy!
>
> So is that a lock conoy then? A bunch of threads waiting on a contended lock? Yes and no. Certainly, highly contended locks do convoy &mdash; but there are subtle, more interesting situations that lead to convoys in lightly contended locks.
>
> Another thought experiment. Like the request processor from before, but now it's a lock. Snap your fingers in a rhythm. Every snap, the thread that has the lock releases it, and at the exact same time, some other thread tries to acquire the lock &mdash; successfully. Your lock is highly utilized, but experiences almost no contention. Everything is good.
>
> Now imagine this: a thread, while holding the lock, "goes out to lunch," so to speak. Maybe it does a file, or network I/O while holding the lock; or maybe you just get plain unlucky and its scheduling quantum ends. Either way, the thread holding the lock goes to sleep without relinquishing it, causing it to hold on to the lock maybe, 100-1000x longer than is typical.
>
> While the thread is asleep (but still holding the lock!) more threads show up and, finding the lock is not available, go to sleep in its wait queue. A convoy has formed. All of a sudden, your normal, run of the mill, lightly contended lock has a long wait queue. And, as long as there are requests outstanding and threads trying to process them, the wait queue may be self-sustaining.
>
> It gets worse.
>
> Managing that queue of waiting threads is expensive. The problem is that thread states can only be managed in the operating system kernel, and transferring control from a user mode program to the OS kernel is incredibly CPU-intensive and time-consuming. On modern CPU architectures, a user mode program asks the kernel to suspend or notify a thread by preparing a request in memory and then triggering a software interrupt (e.g. using the x86 `int` instruction), knowing the kernel is set up to trap that interrupt and process the user buffer. The hardware machinery behind interrupts are expensive, and furthermore, the kernel-mode trap handler still has to figure out what it's being asked to do *and* make sure the request is valid and well-formed before any of the requested work is carried out.
>
> To mitigate the high cost of managing thread states in kernel-mode, most well-designed locks use a fast-path/slow-path design, where the fast path is implemented as some type of atomic memory write (e.g. compare-and-swap) that can be completed 100% in user mode, and the slow path involves a trip to kernel mode to put threads to sleep or wake them up. Many locks use complex retry schemes to try to maximize how often the fast path is taken, because going to kernel is just so darn slow.
>
> Think about all that in terms of our lock convoy. Remember, usually the lock is completley uncontended and has no wait queue. That means we're using the fast path all the time. But then, one thread blips out for a second, and a permanent convoy forms. Now every thread has to pay the cost of a full kernel mode round trip every time it acquires the lock *and* every time it releases the lock. It's not just latency overhead &mdash; now the lock itself requires a lot more CPU time than it used to. 
>
> I hope you're starting to see now, what was going on in the opening scenario. Starting off, all locks in the system are lightly contended, even if some are highly utilized. Then, blip! A thread holding the lock blips out for a little while, and comes back to find a queue has formed of threads waiting for the lock. Now every operation on that lock takes the slow path, driving up the cost of the lock and impacting the system's throughput accordingly. And, like all the convoys we've discussed so far, this queue is likely to sustain so long as there are more requests to process. From outside the system, this manifests as high throughput then, randomly, lower throughput. Inside, a lightly contended lock has unexpectedly become very expensive.
>
> It gets worse. 
>
> Context switches &mdash; the act of changing which thread is running on a CPU core &mdash; are expensive. First off, you need a user-to-kernel round trip, which we already know is expensive. But thread switches are even more expensive, because each thread starts off with a 'cold' CPU cache, which only warms up (allowing the CPU to reach peak performance) as the thread runs. Your operating system scheduler is thus designed to change threads as infrequently as it can get away with, without impacting system responsiveness.
>
> In scheduler parlance, the "scheduler quantum" is the amount of time a thread is scheduled to run. To restate the above, schedulers try to pick quanta that are long enough to amortize the cost of the initial context switch, while short enough that the system still 'feels' responsive. That way, a context switch that costs (??) microseconds of busywork is amortized over (??) milliseconds of thread runtime, before the scheduler interrupts the thread and switches to a differnet task.
>
> It's a beautiful plan. You know what shoots it all to hell? Locks.
>
> Any time a thread voluntarily puts itself to sleep, it yields the processor before its quantum has elapsed &mdash; *before* the amortization process has completed. The scheduler might think it's paying for (??) microseconds of context switch overhead over (??) milliseconds of runtime, but if you stop running after just (??) microseonds, the context switch overhead balloons to (??)%
>
> Back to that lock convoy. Before, we saw that queue of waiting threads isn't just slow because of the user-to-kernel transitions needed to put threads to sleep and wake them up. Now we know it's even *slower* than that, because the wait queue is forcing threads to give up their scheduling quanta early. Way before the cost of the context switch was paid off! We're forcing really expensive context switches to happen way more often than they're designed to. In a well-tuned system, we should be completing (??) requests per scheduling quantum, but now we're taking at least two quanta per request! This system is spending a significant portion of its time switching threads instead of running any of our code!
>
> It gets worse!
>
> People want locks to be 'fair,' meaning threads all get the same quality of service from the lock. That usually looks like a wait queue that's processed first-in, first-out (FIFO); i.e. the lock is granted on a first-come, first serve (FCFS) basis. However, if the lock is strictly FIFO/FCFS, it'll make all convoy effects even worse!
>
> Remember, lock convoys drive down system throughput by taking CPU cycles away from your code and dedicating them to excessive context switches instead. A FIFO lock will force your threads to context switch every time the lock is needed.
>
> TODO I need to go to bed, this needs some elaboration
>
> Summarize the situation relative to the opening hook. Initially, locks are lightly contended and using the fast path successfully with few trips down the slow path, throughput is high, we're happy. Then, blip, we get unlucky, a thread holding the lock gets scheduled out, and a queue forms. Now the lock is expensive forever because we will need to manage thread wait states in kernel mode and context switch early, potentially on every single lock operation if it's strict FIFO/FCFS. And the situation is self-reinforcing, because the expensiveness of the lock guarantees the queue will remain nonempty. 
>
> A discussion about what can/can't be done and some tradeoffs. I had some notes elsewhere that need to be moved here.
>
> 
