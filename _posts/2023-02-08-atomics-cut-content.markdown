---
layout: post
title: Atomics Cut Content
author: Dave
draft: true
---

> ## Memory Models
>
> Acquire and release are an idiom that appear in *memory models*, including those of C, C++, Rust and some CPUs. But, what's a memory model?
>
> A memory model is a set of guarantees around what happens when your code reads or writes variables in memory. Every programming language has a memory model; usually, these are described in (ostensibly, human readable) documents as part of the formal language specification. Compiler people design, publish and adhere to memory models for the programming languages they compile. CPU designers do the same; after all, the instruction set for a CPU is a programming language too. You don't usually need to worry about memory models, but they're important when you're writing-lock-free code.
>
> That's the official story, anyways. But here's what people are usually too polite to tell you: memory models exist because you're getting a bad deal.
>
> There are design problems in multi-core CPUs that we don't know how to fix, and the buck is being passed to you, the lock-free programmer. We'll get into more detail about this later. For now, know a memory model is essentially a carefully worded contract explaining the terms of this deal. Native languages like C, C++ and Rust generally inherit the memory model of the underlying CPU, so you get exposed to all the crazy things CPUs think it's okay to do to your code.
>
> Acquire and release are idioms that appear in a particular memory model: the one shared by C, C++ and Rust. Originally, acquire/release semantics were an abstraction over different but equivalent behaviors implemented in CPUs, but in a "tail wagging the dog" kind of situation, some CPUs now support acquire/release in the instruction set in order to better support C/C++ code written using acquire/release semantics.
>
> In summary, acquire/release semantics are an artifact of odd decisions made in CPU memory models. If memory worked the way you thought it did, you'd never need to acquire or release anything. Unfortunately, memory doesn't work the way you think it does.



> ## Assembly Lines for CPUs
>
> When American elementary schools cover the history of the industrial revolution, they often include a bit on Henry Ford and the assembly line. The story goes like this: once upon a time, building a car was a long and complex process, requiring someone with extensive training on all sorts of mechnical arts to build a very complex machine. Henry Ford realized that, instead of hiring one expensive person to perform all, say, 200 steps of building a car, he could instead hire 200 people to each perform one step, and have them all work in parallel. The result is an assembly line, where there are always 200 cars in varying degrees of completion. 
>
> The key idea here is that parallelism increases the throughput of a sequential process. Once a car makes it all the way through the assembly line, the next car is 199/200 steps complete; the car after it is 198/200 steps complete; and so on. So, each step of the assembly line, you get a full car; without the assembly line, you'd only get one car every 200 steps. That's a 200x improvement!
>
> CPUs do something extremely similar with how they execute instructions. Like assembling a car, executing a single instruction is a long, sequential process. However, if you break that down into, say, 200 little sub-circuits that each execute a step, and run then sub-circuits in parallel, you can make progress 200 instruction each step, exactly like the assembly line was working on 200 cars at a time. Now you complete one instruction each step, instead of one instruction every 200 steps.
>
> When applied to CPUs, this technique is called **pipelining**; pipelining is a form of **instruction-level parallelism**. All modern CPUs are pipelined; it's what makes them fast.
>
> Of course, pipelines aren't a silver bullet. (But what is?) Whether you're designing the CPU or programming one, the key ensuring a pipelined CPU is fast is making sure nothing **stalls** the pipeline. Any time you have to wait for something is a lost opportunity to make progress; the more you wait, the less you get done. Duh! But it's easier said than done. For example, pipelining has a huge effect on how you think about memory. 
>
> ## Memory &amp; Pipelining
>
> Say you're designing a pipelined CPU. One of the big questions you're going to have to figure out along the way is what you do about memory. Memory is slow, and instructions that read or write memory can thus stall the pipeline for lengthy periods of time; however, programs use memory a lot, so you can't really just accept that memory access always stalls the pipeline. If the pipeline is going to be stalled that much, that often, there's little reason to bother pipelining in the first place.
>
> The de facto standard solution to this problem is the **CPU cache**. Essentially, you just put a little bit of very fast memory on the CPU. I won't belabor this point; everyone knows what a cache is, and I'm sure you've heard of CPU caches before.
>
> Without caches, the CPU and memory diagram for a computer looks very simple:
>
> > Diagram with a CPU connected to memory
>
> With a single-core CPU that has a cache, the diagram looks like this:
>
> > Diagram with a CPU through memory
>
> We want to go to multi-core CPUs:
>
> > Diagram with multiple CPUs accessing memory
>
> What should a CPU cache look like in this situation? Should there be one cache per CPU?
>
> > Diagram with one cache per CPU
>
> Or, should we have one shared cache that all CPUs use?
>
> > Diagram with a single shared cache
>
> For correctness reasons, we want to have a single shared cache, like the last diagram. Unfortunately, such a cache would be too slow for practical use. Instead, we need at least some per-CPU cache. The option that most CPU designers go with is a hybrid approach:
>
> > Diagram with per CPU L1 and shared L2,3+



> ## Atomics
>
> An "atomic" update is one where concurrent readers see the update completely or not at all; readers should never see intermediate state or garbage values. If a thread reads a variable in parallel with an atomic update, the reader should see the old or the new value, but never anything else.
>
> What do I mean by anything else, exactly? Consider what would happen in the following scenario, where memory updates are <u>not</u> atomic. Say I had a 16-bit variable with a current value of zero; that is, 16 bits of all 0s:
>
> <center><pre>00000000 00000000</pre></center>
>
> Say a thread running on one CPU core is trying to update the variable to 16 bits of all 1s:
>
> <center><pre>11111111 11111111</pre></center>
>
> Consider what would happen if this write happens one bit at a time, or one byte at a time. If another thread running on a different core tried to read the value in the middle of the update, it might see the following partially updated-value:
>
> <center><pre>11111111 00000000</pre></center>
>
> This would be an example of a **torn read**: the value being read is partially the old value and partially the new value. If the update were atomic, it'd be impossible to ever see this partially updated state, because by definition an update is "atomic" if a reader can only see the old state (`00000000 00000000`) or the new state (`11111111 11111111`).
>
> Fortunately, torn reads aren't really a thing on modern CPUs. On all modern CPUs, loads and stores to a single integer variable are atomic by default (modulo word size and alignment problems beyond the scope of this blog). Usually, when people talk about atomics, they mean **atomic read-modify-write** operations.
>
> Say, for example, that you're trying to maintain a multithreaded counter:
>
> ```c++
> static int g_counter = 0;
> void incr_counter() { g_counter++; }
> ```
>
> If you're calling `incr_counter()` on multiple threads, this will not 'just work' the way the earlier example did. The problem is that `g_counter++` is actually syntactic sugar for three steps:
>
> ```c++
> int temp = g_counter; // read
> temp = temp + 1;      // modify
> g_counter = temp;     // write
> ```
>
> Per our discussion, the read and the write are each atomic, *individually*, but the three step read-modify-write are not atomic as a whole. It's possible for two threads to read the same value, do the same modification, and then both write the same value; if that happens, one of the counts is lost.
>
> > So atomic intrinsics that read-modify-update by locking the memory location
>>
> > It's important to realize atomics only mean "all or nothing"
>>
> > Nothing about order or immediacy.
> 
> 



> ## Atomics don't really help
>
> It's important to understand that atomic memory operations don't really help us fix Peterson's lock algorithms. We can use something like compare-and-swap to *replace* the lock algorithm wholesale, like this:
>
> ```c++
>class TestAndSet : public Lock {
> 
>std::atomic_bool locked;
> 
>public:
> 
>void lock() {
>  while (locked.compare_exchange(false, true))
>      ; // wait
>   }
>   
>   void unlock() {
>  locked.store(false);
>     }
>   };
>    ```
>    
>   But this isn't Peterson's algorithm; it's something else entirely!
>   
>   There's an important distinction to be drawn here between **atomicity** and **ordering guarantees**. Atomic read-modify-write operations like compare-and-swap, fetch-and-add and the like guarantee that updates to a single variable cannot be interrupted, but they don't provide guarantees about the relative order of loads and stores to *different* memory locations. The problem we have with Peterson's lock is 
>    
>   
> 
> 
>
> 
>
> > I've refactored the post again
>>
> > What do I want to say here?
>>
> > How do I broach the idea of ownership and handoff?
>
>  



> 
>
> A-ha! That's how the consumer managed to dequeue null! There was no race in our original code, but the CPU changed our code in a way that introduced a data race. Mystery solved.
>
> > Mystery solved? Or did we just replace one WTF with another, much bigger WTF? How could someone *seriously* think it's okay to ship a CPU that breaks programs by design? Well, read on ...
> >
> > Once upon a time, we just had single-core CPUs. Year after year, CPU vendors came up with new tricks to run code faster and faster, like the "working ahead" trick we just saw. On the original single-core CPUs, these tricks were completely sound: it was impossible for them to break programs. But one year, we hit a wall, and then it was no longer feasible to make a single CPU core faster. Instead, we started adding more cores. But, as you saw, some of the tricks that worked on a single-core CPU break code on these new multi-core CPUs. What *should* we have done about this?
> >
> > Abandoning all the old tricks is the wrong answer &mdash; that gives consumers a no-win choice between a fast single-core CPU or a slow multi-core CPU. So, fine, we shipped multi-core CPUs with all the old tricks left in. Alas, multi-core CPUs break multi-threaded code, and you and I are left to figure wat to do about it. ¯\\(ツ)/¯
>
> So then ... if memory isn't sequentially consistent, what is it? The answer is, *almost* sequentially consistent.
>
> * 

> ## The Golden Rule of Reordering
>
> Because all the cool tricks we came up with for reordering program statements came from a time when CPUs only had one core, all mainstream CPUs adhere to a single commandment:
>
> <center><i>Never change the result of single-threaded code.</i></center>
>
> For example, let's say you had a toy program like this:
>
> ```c++
> int x = 1;
> x = 2;
> printf("%d", x);
> ```
>
> What are some things a CPU can and can't do with this code?
>
> Can we change the program to the following?
>
> ```c++
> int x = 2;
> printf("%d", x);
> ```
>
> Absolutely! If a tree falls in a forest and nobody hears it, does it make a sound? If we set x to 1 and don't read it, did we ever set it to 1? The point is, assuming single-threaded code, you don't have a way of knowing whether x was ever 1, so the CPU can safely skip that step.
>
> What about this change?
>
> ```c++
> int x = 2;
> x = 1;
> printf("%d", x);
> ```
>
> This doesn't work at all, does it? The original snippet would have printed 2; this prints 1. You can't change the meaning of a single-threaded program.
>
> Putting those two examples together, we can start to make broader statements. CPUs generally will not change the relative order of reads and writes to a single variable, other than maybe to skip updates that provably are never read back.
>
> What if we add another thread to the mix? Say the program looked like this instead:
>
> ```c++
> static int x;
> 
> void thread1() {
>   x = 1;
>   printf("%d", x);
> }
> 
> void thread2() {
>   x = 2;
>   printf("%d", x);
> }
> ```
>
> What ordering guarantees do we provide now?
>
> Well, each thread routine reads its own write, so a CPU cannot change the body of either of the thread routines without impacting correctness. In that sense, there's no reordering possible.
>
> However, since both methods can execute concurrently on different CPU cores, there are several possible orderings by which the two methods can be interleaved. Here are a few possibilities (but not all of them, for brevity):
>
> ```c++
> thread1: x=1
> thread1: printf("%d", x)
> thread2: x=2
> thread2: printf("%d", x)
> // output: 12
>   
> thread2: x=2
> thread2: printf("%d", x)
> thread1: x=1
> thread1: printf("%d", x)
> // output: 21
>   
> thread1: x=1
> thread2: x=2
> thread1: printf("%d", x)
> thread2: printf("%d", x)
> // output: 22
>   
> thread1: x=1
> thread2: x=2
> thread1: printf("%d", x)
> thread2: printf("%d", x)
> // output: 11
> ```
>
> We need to stop here for a second so we can call out an important assumption baked into what we just did: if it's valid for us to interleave the steps into a single order like this, then each step must have been *atomic*.

> ## Atomics and Memory Ordering
>
> At a basic level, "atomic" just means "cannot be interrupted." If two atomic operations execute in parallel, the hardware must serialize them: one operation goes first, and the other must either wait or abort.
>
> Usually, when we talk about atomic operations, we think of read-modify-write primitives like compare-and-swap or fetch-and-add. But on mainstream CPUs, even plain memory reads and writes are atomic. That is, CPUs do not allow a read to 'interrupt' a write: if one core wants to read a variable and another core wants to update the same variable, the CPU has to hold one until the other finishes.
>
> > Why do all CPUs provide atomicity guarantees for plain reads and writes? Because it'd be utter chaos if they didn't. For example, say we have a 16-bit variable initially set to zero; that's 16 bits of 0s:
> >
> > <center><pre>00000000 00000000</pre></center>
> >
> > Say one thread is trying to update this variable to 16 bits of 1s:
> >
> > <center><pre>11111111 11111111</pre></center>
> >
> > What if another thread is trying to read this variable as the update is being carried out? If there's no atomicity guarantee, then the reading thread could see the value of the variable before it's completely updated. For example, say the updating thread has to update the variable one byte at a time; then the reading thread could see this:
> >
> > <center><pre>11111111 00000000</pre></center>
> >
> > This is an example of *tearing* (pronounced "tare-ing" as in ripped, not "teer-ing" as in crying; although, if tearing bugs bring you to tears, I get that.) Having to deal with tearing would make it nearly impossible to write correct lock-free code, so CPU vendors did us a solid and promised not do this. Loads and stores are atomic, so you always see the variable as it was before or after the update; never intermediate state and never random garbage.
>
> But here's the weird thing: if all reads, writes, and read-modify-writes to a single variable are serialized this way, then that implies there's a single global order for all reads and writes to a single variable. Reads and writes to a single variable are sequentially consistent! That's the reason we were able to write interleavings of `thread1()` and `thread2()` earlier.
>
> Hang on, though &mdash; so is memory sequentially consistent, or isn't it?
>

> ## What we don't have: Linearizability
>
> So if we have sequential consistency for a single thread with multiple variables, and we also have sequential consistency for multiple threads using a single variable, what goes wrong when we try to go to multiple threads and multiple variables?
>
> The problem is the per-variable orders don't compose. They may work individually, but you can't combine them into a single global order that makes sense.
>
> TODO linearizability actually matches the debugger model. It requires there to be a single point during the load, store or rmw at which the change becomes globally visible to all threads. That's what you think is happening in the debugger model, and if you did it, you'd get 
>
> TODO what goes wrong is a core can 'work ahead,' continuing to work on a variable that's still stuck in a local cache or buffer. It makes progress on that variable and can see its own writes to that variable, but nobody else can see writes to that variable



> Here's the idea: if memory is *linearizable*, then it would be possible in principle to get out a stopwatch and record three points in time for each instruction that reads or writes a variable in memory:
>
> 1. A start time, when the instruction began execution
> 2. An end time, by which the instruction had finished
> 3. A commit time, at which the change takes effect on all CPU cores
>
> Additionally, these times must satisfy two more constraints. First, the commit time for an instruction must always fall between the instructions start and end times; that is, changes take effect globally *while the instruction is executing*. Second, the instructions must be sequenced in program order. So if your program goes
>
> ```c++
> x = 1;
> y = 2;
> ```
>
> ... then the end time of the `x = 1` instruction must come before the start time of `y = 2`. 
>
> > What I said is still a little imprecise and incomplete, we really want to get at the idea of a linearization point, but I think I'm cutting this in-depth definition anyways so we'll see.
>
> 
>
> 
>
> 
>
> 

> So what is memory, if not sequentially consistent? Something very close to it, in fact:
>
> * Memory *appears* sequentially consistent **for a single thread**
> * Memory *actually is* sequentially consistent **for a single variable**
>
> It's only when we have **multiple variables shared by multiple threads** that we run into trouble. And even then, CPUs give us tools to help in these situations &mdash; tools like the acquire-release fences mentioned at the start of this post. But before we jump into those, let's take a quick look at what I mean by single-thread and single-variable consistency.
>
> ## Sequential Consistency in One Thread
>
> Although CPUs don't always read and write memory in the order your code says to, they go to great lengths to make it look *as if* memory reads and writes are always in same order as your code says to &mdash; as long as all your memory is local to just one thread.
>
> To understand why, you have to understand how we ended up with messed up memory semantics in the first place:
>
> Once upon a time, we just had single-core CPUs. Year after year, CPU vendors came up with new tricks to run code ever faster, like the "working ahead" trick we saw before. On single-core CPUs, these tricks were sound: they could never break programs. But one year, we hit a wall, and then it was no longer feasible to make a single CPU core faster. Instead, we started adding more cores. But, as you saw, some of the tricks that worked on a single-core CPU break code on these new multi-core CPUs. What *should* we have done about this? Abandoning our tricks is the wrong answer &mdash; you don't want multi-core CPUs to have poor single-thread performance. So, we ended up with multi-core CPUs that still do all the old tricks, even if that sometimes messes up multithreaded code.
>
> The thing is, since the old tricks were always sound on *single-core* CPUs, they all happen to obey one central commandment:
>
> <center><i>Never change the outcome of single-threaded code.</i></center>
>
> Even as we moved to multi-core CPUs, we've kept this rule intact.
>
> For example, let's say you had a toy program like this:
>
> ```c++
> int x = 1;
> x = 2;
> printf("%d", x);
> ```
>
> According to the central commandment, a CPU is allowed to change our program oto this:
>
> ```c++
> int x = 2;
> printf("%d", x);
> ```
>
> If a tree falls in a forest and nobody hears it, does it make a sound? If we set x to 1 and don't read it, did we ever set it to 1? The point is, assuming single-threaded code, you don't have a way of knowing whether x was ever 1, so the CPU can safely skip that step.
>
> Could we do this, instead?
>
> ```c++
> int x = 2;
> x = 1;
> printf("%d", x);
> ```
>
> No way, right? The original program would have printed 2; this prints 1. This change has never been acceptable, then or now.
>
> The point of these examples is there are limits to how CPUs can change your code at runtime; the inside of a CPU is not such a lawless place. Because the CPU will never change the outcome of your code *from the current thread's point of view*, it follows that you can get away with assuming sequential consistency, or the 'debugger model,' so long as you're sharing your memory with other threads. We can only "see" that code was reordered from the outside, by viewing the effects from another thread. Inside of the current thread, it all 'just works.'
>
> ## Sequential Consistency in One Variable
>
> But what if we *do* have multiple threads? It turns out, we still get sequential consistency as long as the threads only communicate through a single variable.
>
> > When I say "one variable" in this blog post, what I technically mean is "single word of data stored in memory at a word-aligned address." If that's gibberish to you, don't worry: the compiler takes care of most these details for you, and you can think of "one variable" as meaning one int, long, char, bool, pointer, etc.
> >
> > Things that do not count as "one variable" include strings, arrays, and structs that have more than one field, although a *pointer* to any of these things is "one variable." Also, big integers like int64 on a 32-bit processor, or int128 on a 64-bit CPU, do not count as "one variable" either.
>
> Single-variable consistency is guaranteed because all memory operations on a single variable are *atomic*. Here, "atomic" just means that an operation cannot be interrupted. If two atomic operations execute on the same variable in parallel, the hardware must serialize the operations: one goes first, and the other must either wait or abort.
>
> Usually, when we talk about atomic operations, we think of read-modify-write primitives like compare-and-swap or fetch-and-add. But on mainstream CPUs, plain memory reads and writes are also atomic. For example, that means a read is not allowed to 'interrupt' a write: if one core wants to read a variable and another core wants to update the same variable, the CPU holds one until the other is finished. (Under the hood, all this is accomplished using cache coherency protocols that are beyond the scope of this post.)
>
> This leads us to a subtle but very important point. Sequential consistency means it is *possible* to construct a serial order of reads, writes, and read-modify-writes that explain what happened to that variable; but if all those operations are atomic, then the hardware is forced to *actually serialize those operations*. So, they're trivially sequentially consistet, because they're actually sequential.
>
> Huh.
>
> So, wait, if the hardware is providing sequential consistency guarantees already, why was our queue implementation broken? Well, these sequential consistency guarantees hold for a *single* variable, but they don't hold *across* variables. There's a nice, clean, sequentially consistent history of updates for `tail`, and another one for `entries[i]`, but it's possible (e.g. if our CPU did its working-ahead trick) that those two histories are not compatible with one another &mdash; there's no way to combine them without changing the order things happened.

> ## Acquire and Release Semantics
>
> ---
>
> Better wording:
>
> What memory ordering guarantees do threads need when they're doing this ownership handoff thing? Hopefully something less than a full `fence()`?
>
> ---
>
> This two-phase handoff pattern where threads acquire and release temporary ownership of something in shared memory, is a common scenario we'd like to optimize. The question is, how? When we do this kind of handoff, where do we need fences, and what's the bare minimum constraint these fences need to impose?
>
> Let's start with the **release** side of handoff, where a thread relinquishes its temporary ownership of memory. For example, consider `toy_queue::put`, which releases ownership of the item:
>
> ```c++
> void put(void *entry) {
>   unsigned i = tail % size;
>   entries[i] = entry;
>   // release fence goes here
>   tail++;
> }
> ```
>
> What's the minimum amount of work we'd need to do in the release fence here?
>
> <!--
>
> I got kind of stuck here trying to explain acquire and release fences, which have very tricky semantics and aren't very useful or the most efficient route. I think the best bet is to completely ignore acquire-release as fence semantics and start thinking of them as semantics for a read, write, or read-modify-write to a variable. That's sufficient for every use case we've discussed so far anyways, and maps better to the C++ atomics library.
>
> So what we need to do is change the setup not to say "a better kind of fence" but rather "something better than a fence" and then get here to the idea of memory ordering around an operation, rather than a fence, and having half-fence semantics, where releases only affect operations before the write and acquires only affect operations after the read.
>
> Oh, and here's the intuition behind read-acquire and write-release:
>
> * If you can write to acquire ownership, without checking anything, then you already had ownership in the first place!
> * If you can release ownership without writing anything, then you never had ownership in the first place!
>
> Oh, and finally, we need to be clear when we first describe `fence()` and fences that this is just one of the many possible semantics of a fence. And later, we should clarify I described acquire-release semantics, and hint there are even stronger synchronization guarantees. 
>
> ---
>
> We may need to do some extra setup here because I feel we're not ready to jump in yet.
>
> First of all, we need to zoom out to the parent program and note how we need to settle all reads and writes that happened before put, as well as any reads and writes that happened inside put, before incrementing tail. Similarly, no reads and writes that happen after poll can start until poll got an item.
>
> Secondly, we need to set up for why some operations can be moved and others cannot. We need to show how write-releases need to stay on the other side of the fence, but following reads can pass through safely; similarly, we need to show how read-acquires need to stay on the oter side of the fences, but preceding writes can pass through safely.
>
> Finally, we need to an intuitive justiication for acquires always being reads. Even I'm confused now on why these semantics are sufficient for taking a lock? I mean, it's convenient that a read-modify-write can always serve as an acquire-read which makes this viable for locking, but we need to find a much more intuitive justification for why it works this way. Why can't you have an acquire-write for example?
>
> Maybe it's good enough to handwave and always say "you see you have access" either by a plain load like in the queue, or with an atomic read-modify-write like in a lock, but "seeing" that your write succeeded feels like a kind of dumb explanation.

> ##  What's it like to get fenced?
>
> Imagine you're working on a project,  you're trying to get a big release out the door; there's a bug bash, people are triaging bugs and forking them over to devs using Jira or something; you're watching your tickets and working through them as they arrive. You code valiantly, squashing bug after bug and putting out fix after fix. Eventually, your queue starts to look pretty empty: you still have several bugs assigned to you, but they're all in various stages of code review, CI validation, etc. You spend the next day or so working less hard, shepherding your fixes into the release branch one by one, but otherwise taking your time, taking long lunches, taking walks, etc. Life is good, eh!
>
> A day or two later, you finish shepherding the last of your bugfixes into the release branch, and you close Then suddenly, *pop!* Several dozen new bugs instantly appear all at once! What? Your boss drops by to ask how it's going, and you're embarassed to say you know nothing about all these bugs on your plate.
>
> What happened? Turns out someone assigned you an obscure kind of Jira ticket called a "fence." The fence blocked your progress, making all new tickets invisible until you had completely cleared out all your existing tickets. How obnoxious! It's nice to take a break once in a while, but this was one of those times where deadlines mattered, and you spent a bunch of time doing nothing because you didn't know there was more work in the pipeline. Had you known, you could have started some of these bugs sooner, and now you have to stress out to catch up. Sucks, huh? That's how your CPU feels about fences.
>
> Just as triaging, designing, coding, validating, reviewing and merging a fix is a whole production for you, so too is executing a single assembly instruction a whole production for a CPU. And just as you can often make the work go faster by breaking it down and doing steps out of order, so can the CPU finish your code sooner by breaking down and reordering instructions. But fences mess that up. They force your CPU to completely close out (or "retire") all in-progress instructions before even seeing more instructions. That means more dead time, and dead time means slowdowns.
>
> The moral of this story is that **more reordering is more better**.
>
> The more you constrain a CPU, the less ability it has to rearrange its work so it gets done sooner; so, the slower things get done. So, the more often you use fences, the slower your CPU runs in practice. In fact, that's why CPU vendors stubbornly refused to give us linearizable memory in the first place, no matter how nicely we ask: it'd be the CPU equivalent of only letting a developer only see one Jira ticket at a time!
>
> In this new light, maybe the `fence()` method is overkill for the situation we're in. Can we do something else, something with weaker (meaning more flexible) memory ordering guarantees, without breaking our queue? The meta-gamer in you knows I'm about to tell you the answer is yes. Let's think about a very common pattern for sharing memory in multithreaded code:

> ## Advice and Takeaways
>
> So how do we use memory ordering guarantees in our code? In the end, you ideally want to end up with the weakest memory ordering guarantees that don't break your code. Here are some examples:
>
> Obviously, any variable that isn't shared should be relaxed: either make it a stdatomic type for which you always specify `relaxed` memory ordering, or even better, just make it as a plain C/C++ variable.
>
> This advance also applies to variables which happen to be stored in shared memory, but in practice can only be used by one thread. The toy queue's `head` is an example of this, as it's only ever read or written by the consumer thread:
>
> ```c++
> bool poll(void **entry) {
>   unsigned t = tail.load(memory_order::acquire);
>   if (t > head) {
>     unsigned i = head % size;
>     *entry = entries[i];
>     head++;
>     return true;
>   } else {
>     return false; // empty
>   }
> }
> ```
>
> Atomic read-modify-write operations can often be relaxed too, if its updates don't need to synchronize with anything else. A common example is a standalone counter that just tracks the number of times your program did something; you can often tag these as `relaxed` so they imposes no ordering requirements on the surrounding code:
>
> ```c++
> atomic_uint lucky_sevens;
> // ...
> if (user_id % 1000 == 777) {
> 	lucky_sevens.fetch_add(1, memory_order::relaxed);
> }
> ```
>
> This tends to work well for tracking observability metrics.
>
> Finally, you frequently find that operations can be relaxed if you just did a read-acquire, or you know a write-release is coming. For example, our `put()` code doesn't need to impose any memory ordering guarantees when we update `entries[i]`, because we know the upcoming write-release will ensure the update is visible to the consumer:
>
> ```c++
> void put(void *entry) {
> 
>   unsigned t = tail.load(memory_order::relaxed);
>   unsigned i = t % size;
>   entries[i] = entry;
> 
>   tail.store(t + 1, memory_order::release);
> }
> ```
>
> Of course, not every operation can be relaxed. If you have a single variable update that gates visibility of some update across threads, you need the update to be a `release` and the check to be an `acquire`, whether that check is passive (such as `toy_queue::poll()` reading the queue tail) or active (such as `toy_lock::try_lock` compare-and-swapping the lock state).
>
> Finally, if you're ever not sure what level of guarantees you need, you can always start by specifying `seq_cst` ordering guarantees and relaxing them later. In fact, that's the direction stdatomic will push you by default: if you *don't* specify a memory ordering requirement, you get sequential consistency. If you go down this route, then once you have code that works, you can experiment with weaker ordering. Just be careful not to fall into the guess-and-test trap, lest you end up with buggy code that happens to work on your CPU but somebody else's!

> ##  What's it like to get fenced?
>
> Imagine you're trying to get a big release out the door; there's a bug bash, people are triaging bugs and forking them over to devs using Jira or something; you're watching your tickets and working through them as they arrive. You code valiantly, squashing bug after bug and putting out fix after fix. Eventually, your backlog starts to look pretty empty: you still have several bugs assigned to you, but they're all in various stages of code review, CI validation, etc. You continue to shepherd your fixes through these processes, but you're now working less hard, as there aren't any more bugs for you to work ahead on. Life is good!
>
> A day or two later, you close out your very last bug, and then *pop!* A dozen new bugs instantly appear all at once! What? As if clairvoyant, you boss drops by seconds later to ask how all those bugs are going, and you're forced to admit you know nothing about any of them, let alone had you been working on them. 
>
> What happened? Turns out someone accidentally assigned you an obscure kind of Jira ticket called a "fence." The fence made all new tickets invisible to you until you had completely cleared out all your existing tickets. That's a shame, because you had plenty of spare time to start working on those bugs, if you had known they even existed. But now you're behind schedule. Sucks, huh? Well, that's how your CPU feels about fences too.
>
> Just as triaging, designing, coding, validating, reviewing and merging a fix is a whole production for you, so too is executing a single assembly instruction a whole production for a CPU. And just as you can often make the work go faster by breaking each fix down and doing the little steps out of order, so too does the CPU finish faster if it's allowed to break down and reorder the little steps of your program's instructions. But fences mess it all up. They force your CPU to completely close out (or "retire") all in-progress instructions before seeing what's next. Artifically preventing the CPU from keeping its pipelines full slows it down.
>
> The moral of this story is: **more reordering is more better**.
>
> The more you constrain a CPU, the less ability it has to rearrange its work so it gets done sooner; so, the slower things get done. That's exactly what happens each time you use a fence. It's also why CPU vendors stubbornly refuse to give us linearizable memory no matter how nicely we keep asking, or how much money we offer to throw at them: it'd be the CPU equivalent of only letting you see one Jira ticket at a time!
>
> In this new light, maybe the `fence()` method was overkill for what we're trying to do. Can we do something else, something with weaker (meaning more flexible) memory ordering guarantees?
>
> Well, it's tricky. There are a lot of things you can do in hardware *in principle*, but you only have space for so many transistors, which means anything you're going to put in hardware had better be for something a lot of software can use. Anything we invent here has to work for lots of multithreaded programs, not just our queue. Luckily, our queue happens to use a *very* common pattern we can certainly optimize for:

> ## In Summary
>
> So, what have we learned?
>
> For one, we learned that **CPUs change our code in the name of optimization**. Just like how you sometimes interleave different pieces of Jira tickets or work items or whatever so that you spend less time waiting and get stuff done faster, so too will CPUs break down and interleave different pieces of your program's assembly instructions in the name of getting stuff done faster. We learned that these changes are always "innocuous" from a single-threaded perspective, but sometimes, you can see their effects in multithreaded code. 
>
> Then we learned **we can set limits on how the CPU changes our code**. We described a hypothetical `fence()` method that enacts a barrier within our code that no instruction can cross. If a CPU wants to pull up an instruction so it happens earlier, or push it out so it happens later, it can do so long as the instruction never crosses a call to our `fence()` method. However, because fences force CPUs to spin down and back up again, they come with a performance cost. We set out to try to find something more flexible (less performance impact) than `fence()`.
>
> That's when we came up with **acquire-release semantics**. These work any time we have shared memory that belongs to one thread at a time, and we want to hand off ownership of that memory from one thread to another. We found that the handoff protocol consists of two steps:
>
> * A **release operation** which is a write to one variable, somewhere in memory, indicating the memory is no longer owned by the calling thread. This write has **write-release semantics**, which means it cannot happen until all preceding memory changes have been carried out; however, the CPU may work ahead on something else if it so wishes.
> * An **acquire operation** to complete the handshake, which includes some kind of read to observe the write described in the previous bullet. This read has **read-acquire semantics**, which means the CPU cannot work ahead until this read has completed; however, the preceding operations do not need to be closed out before the read, so the CPU can delay work if it so wishes.
>
> Finally, it turns out these acquire-release semantics are a natural fit for **locks, queues, and a variety of threading primitives**. Any time you're handing off memory from one thread to another, where the ownership change is synchronized using changes to a single variable, you can almost certainly use acquire-release semantics to ensure handoff is clean:
>
> * The new owner sees everything the previous owner did before releasing it
> * The previous owner cannot use the memory after releasing it
>
> <div class="tenor-gif-embed embedded-figure" data-postid="10381649" data-share-method="host" data-aspect-ratio="1.37" data-width="100%"></div> <script type="text/javascript" async src="https://tenor.com/embed.js"></script>
>
> It's been a long journey, but hopefully you now have a better handle on memory ordering and ordering semantics!
