---
layout: post
title: Using Acquire and Release Semantics
author: Dave
draft: true
---

"Memory ordering semantics" like *acquire* and *release* are confusing. Plenty of people who are perfectly competent at writing lock-free code in native languages like C, C++ or Rust, still have trouble understanding what these things do or when they're helpful. I'm hoping this post can help clear up some of the confusion.

I think you'll know everything you need about these things once you can answer this question:

<center><i>What</i> is being acquired and released?</center>

The short answer is *ownership of other memory*. But I'm guessing that leaves you with more questions than answers. Let's back it up ... a lot. In figuring out how these acquire-release things are supposed to help us, we might find ourselves questioning our most basic assumptions of  how computers run code. It'll be a long climb, but I think you'll like the view from the top!

Read this in your best [Gilfoyle](https://www.hbo.com/silicon-valley/cast-and-crew/bertram-gilfoyle) impression: *If memory worked the way you thought it did, you'd never need to acquire or release anything. Unfortunately, memory doesn't work the way you think it does.*

## How you think memory works

For the sake of example, say you're building a lock-free single-producer single-consumer ring queue. In more words: we're building a data structure with queue semantics, backed by a ring buffer, where we have one thread that enqueues all the items, and another thread that dequeues all those items asynchronously. If this seems contrived, just pretend we're working with Linux's [io_uring](https://en.wikipedia.org/wiki/Io_uring) interface, where a ring is two of these queues: one to submit I/O from userland to the kernel (the send queue) and another for the kernel to deliver results back to the user (the completion queue).

Here's a 'toy' implementation of such a queue:

```c++
class toy_queue {
  
  unsigned head;
  unsigned tail;
  unsigned size;
  void **entries;
  
public:
  
  void init() {
    head = 0; tail = 0;
    memset(entries, 0, size * sizeof(void *));
  }
  
  void put(void *entry) {
    unsigned i = tail % size;
    entries[i] = entry;
    tail++;
  }
  
  bool poll(void **entry) {
    if (tail > head) {
    	unsigned i = head % size;
    	*entry = entries[i];
    	head++;
    	return true;
    } else {
      return false; // empty
    }
  }
  
};
```

We will be iterating on this code throughout the post, so make sure you get it before moving on. All good? Then here's a puzzle for ya:

There's a bug in this queue, and I'm wondering if you can find it. When you run this on a real computer, sometimes `poll()` returns true without giving you an entry. That is, `poll()` returns true, but `entry` is still null &mdash; even though the producer enqueued something non-null. Here's a repro program that sometimes demonstrates the issue:

```c++
static toy_queue queue;
static foo an_entry;

void producer() {
  queue.put(&an_entry);
}

void consumer() {
  void *entry = nullptr;
  bool exists = queue.poll(&entry);
}
```

Since we're letting the producer and consumer race, there are two valid outcomes for the consumer:

1. `exists == false` and `entry == nullptr` (poll serialized before put)
2. `exists == true` and `entry == &an_entry` (put serialized before poll)

However, the bug I want you to puzzle over, is that *sometimes*  `exists == true` and `entry == nullptr`. How could that happen?

Reread the code and think it over!

If you think it's obvious this can happen, try working backwards. If `poll()` returns true, then it must have seen `tail > head`, which means `put()` had already incremented `tail` beforehand. But incrementing `tail` is the *last* thing that happens in `put()`, and checking `tail` is the *first* thing that happens in `poll()`. So it seems `put()` finished before `poll()` even started. But that would mean the two methods ran sequentially, with zero parallel overlap. So where did the entry go?

Just to show you there's nothing up my sleeve, let's take a closer look at how the two threads could have interleaved. The following order should be the only possible way to interleave the two threads' logic; notice how moving the first line of `poll()` any earlier would have changed its return value:

```c++
put: unsigned i = tail % size
put: entries[i] = entry
put: tail++
poll: if (tail > head) {
poll: unsigned i = head % size
poll: *entry = entries[i]
poll: head++
poll: return true
```

But this clearly has `put()` writing to `entries[i]` before `poll()` reads it. So how could it still be null?

See, not so obvious! If I've managed to stump you now, then allow me to reveal what's wrong. We fell victim to one of the classic blunders, the most famous of which is "never get involved in a land war in Asia," but only slightly less well-known is this:

## Memory isn't linearizable.

Have you used a debugger before? If yes, I suspect it affected your intuition of computer memory. If you've ever stepped through code line by line, watching as your variables change each time you step, it's tempting to think that's how code really works: running one line at a time (or at least, one assembly instruction at a time), with the contents of your variables changing in memory instantly during each step.

We can say this more rigorously than "like in a debugger." In fact, there's a theoretical framework that captures this intuition with formal mathematics, called **linearizability**. The basic idea is that, if your computer guarantees linearizability, then you should always be able to identify a single point in time during each instruction when the memory changes for that instruction became visible to every thread simultaneously. That point in time is called a *linearization point*.

If you can collapse each thread's logic into a sequence of linearization points, it's formally proven you can do something that intuitively seemed right all along: combining the linear histories of two threads into a single linear history that explains what happened. Sound familiar? Because, that's exactly what we just *failed* to do for our repro program! The reason is simple: there's no such thing as a single global order of memory operations on a real computer, memory isn't linearizable, and what you see in a debugger isn't what happens at runtime!

![](/assets/debuggers-lie.jpeg)

Take a closer look at the body of `toy_queue::put()`:

```c++
unsigned i = tail % size;
entries[i] = entry;
tail++;
```

Remember, your CPU doesn't directly read and write memory; memory is many times slower than a CPU, so the CPU reads and writes all memory through a cache. Maybe when running this code, it so happens that we get a cache hit for `tail` and `size`, but we miss on `entries[i]`. Now we're blocked: we can't execute the second line of this function until the memory contents come into the cache, and since memory is so slow, that might take a while.

Wouldn't it be nice if the CPU could work ahead while it waits for the data? The people who designed your CPU certainly thought so! How about we execute the third line of code while we're blocked on the second? In other words, why not run the program *as if* it were written like this?

```c++
unsigned i = tail % size;
tail++; // <-- do this early
entries[i] = entry;
```

That's *not* what the code says, but as long as we don't change the outcome, who would ever know the difference? Either way, `tail` gets incremented and `entries[i]` gets initialized; it's not like either value gets read before both are written. So there's the harm in switching the order, right? Right! There are CPUs designed according to this train of thought, and will sometimes make this change to your code, making it faster without impacting correctness.

Hm, but we *did* impact correctness, didn't we? Looking at `put()` narrowly through a single-threaded lens, it doesn't seem like changing the order of these two memory writes matters, but when we zoom out and look at the whole program, it's definitely broken now. This two-line swap introduced a race: if you bump `tail` before writing `entries[i]`, you're reporting there's an item in the queue *before* the item is actually in the queue! So now the consumer might go and read `entries[i]` before you write it!

```c++
put: unsigned i = tail % size
put: tail++               <-- oops
poll: if (tail > head) {  <-- oh no
poll: unsigned i = head % size
poll: *entry = entries[i] <-- stop
poll: head++
poll: return true
put: entries[i] = entry   <-- no
```

Well, that explains it: `poll()` sometimes returns true without an entry, because that's really the state it sees in the queue: `tail` really says there's an entry available, and `entries[i]` really hasn't been initialized with the entry yet. But hey, our code doesn't do that &mdash; the CPU did. This is a highly unusual situation, wouldn't you say? I mean, how many times have we all made fun of this guy?

![](/assets/cpu-is-wrong.jpeg)

Well, this time, Principal [Software Engineer] Skinner might have a point. We wrote correct code, and in trying to dynamically optimize it, the CPU transformed our correct code into broken code. Yet, if we talked to someone who designs CPUs, they would likely turn this around on us: *our* code is wrong because *we* were wrong to assume linearizabile memory. 

![](/assets/code-is-wrong.jpeg)

Who's to blame for this problem is an old argument hashed out many times on remote corners of the Internet (not always productively), and ultimately, what matters is CPUs aren't going to change, which means our code has to.

## So what do we do now?

If you're meta-gaming, you might think this is the point we break out the atomics &mdash; add some fetch-and-adds or compare-and-swaps and watch our problems melt away, right? Not today, I'm afraid.

Atomics are useful for solving a different problem. Atomics resolve conflicts between parallel updates to the same variable. For example, if we had multiple threads trying to increment `tail`, we could use atomics to serialize the updates so each thread's update does something meaningful. But there are no such conflicts in our queue. We have exactly one producer thread managing all updates to `entries` and `tail`, and one consumer thread managing `head`. If all updates to any one variable always come from the same thread, then updates are always serial, so there are no conflicts to resolve. No need for atomics.

Hm, then what exactly *is* our problem? It's **memory ordering**.

When we put an entry in the queue, we need to update *two* variables: the queue entry and the tail pointer. We are certain that order matters: we want to write the entry first and *then* increment the tail, because the latter step is what indicates to the consumer that it's good to start polling. If the consumer polls the queue after we've written the entry but before we've incremented the tail, it passes by without 'seeing' the partially completed put operation. This is all a perfectly good plan.

The only reason this didn't work is we don't have the *memory ordering guarantees* we thought we did. We were very reasonable, but ultimately wrong in thinking writing two lines of code in order would make the CPU execute them in order. Now that we know a CPU might shuffle our code around in the name of making progress faster, we need some way to tell the CPU no, you can't do that, order matters here.

**Fences** are the tool we get for doing that.

Fences are CPU instructions, but as CPU instructions go, fences are a little odd. They don't do anything; they just sit there in your code, marking locations like mile markers on a freeway. We call them"fences" because they enact boundaries. If a CPU decides it's a good idea to push an instruction later, or pull an instruction earlier, but the instruction runs into a fence along the way, then *bonk!* the instruction stops moving. Instructions can't cross a fence.

To see how this works in practice, let's imagine we have a method called `fence()` that gets compiled to a CPU fence instruction. Then here's how we might use `fence()` to fix our queue:

```c++
class toy_queue {
  
  unsigned head;
  unsigned tail;
  unsigned size;
  void **entries;
  
public:
  
  void init() {
    head = 0; tail = 0;
    memset(entries, 0, size * sizeof(void *));
  }
  
  void put(void *entry) {
    unsigned i = tail % size;
    entries[i] = entry;
    fence(); // <-- new!
    tail++;
  }
  
  bool poll(void **entry) {
    if (tail > head) {
      fence(); // <-- new!
      unsigned i = head % size;
      *entry = entries[i];
      head++;
      return true;
    } else {
      return false; // empty
    }
  }  
  
};
```

See that new `fence()` call in `put()`? It fixes our problem. Now if the CPU gets bored waiting for `entries[i]` to be read into the cache, and tries to pull up the `tail++` line so it happens earlier, *bonk!*, the `tail++` line hits that fence and stops moving. We thereby force the entry to be written before the tail is incremented. Problem solved!

But hang on ...

![](/assets/fence-in-poll.jpeg)

There was a second bug all along! 😀 Just as the producer must make sure the entry is written before making it visible (by incrementing the tail), so too must the consumer ensure the entry is visible (tail was incremented) before reading the entry itself. Without that `fence()` call in `poll()`, a CPU would be allowed in principle to change `poll()` to something like this:

```c++
unsigned i = head % size;
void *temp = entries[i];

if (tail > head) {
  *entry = temp;
  head++;
  return true;
}

return false; // empty
```

Once again, this is a perfectly valid transformation for single-threaded code, but it's a disaster for our queue. What if `put()` runs after we read `entries[i]` into `temp`, but before we check `tail > head`? Then we'd read a null entry (`put()` really hadn't started yet) but later see an entry is available (because by then, `put()` had run to completion). This is a new bug with the same symptoms: the return value is true but the entry isn't initialized. That's what the new fence in `poll()` fixes.

Now I don't know about you, but this second bug makes *me* nervous. So far the CPU has come up with two different ways to break our code, and both were the kinds of subtle, impossible-to-repro problems that can slip through review and testing and then run roughshod through production. When is this going to end? How do we know when the CPU is out of tricks to throw at us? Can we ever declare our queue correct?

Seems like a good time to actively prove our queue works.

## Reasoning about Fences

No more shooting from the hip and reasoning based on what we expect. To show the bug is fixed, we need to understand what guarantees the CPU gives us, and use those to justify what we did. We need to that rules-lawyering friend you keep inviting to your DnD sessions for some reason.

There are four steps needed to hand off an entry through the queue:

1. The entry is written to `entries[i]`
2. The entry is made 'visible' by incrementing `tail`
3. On poll, a consumer sees `tail` was incremented
4. The entry is read from `entries[i]`

If all four of these steps run in order, there is no 'missing entry' bug. There can't be: `poll()` only thinks an entry is available in step 3 if step 2 has completed, and when that happens, the entry is read (4) after it was written (1). Any time an entry is available, the corresponding entry's read comes after the write.

For reference, here are `put()` and `poll()` again, with those four steps marked:

```c++
void put(void *entry) {
  unsigned i = tail % size;
  entries[i] = entry; // (1)
  fence();
  tail++; // (2)
}

bool poll(void **entry) {
  if (tail > head) { // (3)
    fence();
    unsigned i = head % size;
    *entry = entries[i]; // (4)
    head++;
    return true;
  } else {
    return false;
  }
}  
```

Do we know the order really is always 1-2-3-4 now? We'll establish this using pairwise relationships:

**The entry is written (1) before the tail is updated (2)**: This order is guaranteed by the fence that appears between these two steps. The fence works because both steps are in the same thread.

**The producer increments the tail (2) before the consumer sees it (3)**: This order is guaranteed by the 'obvious' rule that memory only changes if someone writes it; in this program, only the producer writes to tail. Note we have no guarantee how long it takes after the producer writes the tail (2) for the value to become available for the consumer to read (3); however, this doesn't affect the relative order of (2) and (3).

**The consumer sees the new tail (3) before it reads the entry (4)**: Once again, we have two operations in the same thread that appear on opposite sides of a fence, so they happen in order.

And there you go. Even though the CPU might start step 1 way earlier than you might expect, and even though step 4 might start way late, and even if plenty of time could pass between steps 2 and 3, none of the matters; we've established that the relative order of all four steps will always be 1-2-3-4 using guarantees we have from the CPU. The bug is now definitely fixed! 🎉

As you can see, ordering guarantees like the kind offered by fences are just as useful as atomics for writing lock-free code. Unfortunately, fences come with a cost. To see why, let me spin a little story for you:

##  What's it like to get fenced?

Imagine you're trying to get a big release out the door; there's a bug bash, people are triaging bugs and forking them over to devs using Jira or something; you're watching your tickets and working through them as they arrive. You code valiantly, squashing bug after bug and putting out fix after fix. Eventually, your queue starts to look pretty empty: you still have several bugs assigned to you, but they're all in various stages of code review, CI validation, etc. You continue to shepherd your fixes through these processes, but you're now working less hard, as there aren't any more bugs for you to work ahead on. Life is good!

A day or two later, you close out your very last bug. Then suddenly, *pop!* Several dozen new bugs instantly appear all at once! What? Your boss drops by to ask how all those bugs are going, and you're embarrassed to say you know nothing about any of these, let alone had you been working on them.

What happened? Turns out someone accidentally assigned you an obscure kind of Jira ticket called a "fence." The fence made all new tickets invisible to you until you had completely cleared out all your existing tickets. How obnoxious! I mean, breaks *are* pretty nice, but this was one of those times where you'd rather have kept getting things done. Now you've spent a bunch of time doing nothing because you didn't know there was more work coming down the pipeline; had you known, you could have started some of these bugs sooner. Now you have to catch up. Sucks, huh? That's how your CPU feels about fences, too.

Just as triaging, designing, coding, validating, reviewing and merging a fix is a whole production for you, so too is executing a single assembly instruction a whole production for a CPU. And just as you can often make the work go faster by breaking it down and doing the little steps out of order, so can the CPU eat through your code sooner by breaking down and reordering instructions. But fences mess that all up. They force your CPU to completely close out (or "retire") all in-progress instructions before seeing what's next. That means more dead time, and dead time means your code takes longer.

The moral of this story is: **more reordering is more better**.

The more you constrain a CPU, the less ability it has to rearrange its work so it gets done sooner; so, the slower things get done. So, the more often you use fences, the slower your CPU runs in practice. In fact, that's why CPU vendors stubbornly refused to give us linearizable memory in the first place, no matter how nicely we keep asking: it'd be the CPU equivalent of only letting you see one Jira ticket at a time!

In this new light, maybe the `fence()` method is overkill for what we're trying to do. Can we do something else, something with weaker (meaning more flexible) memory ordering guarantees? The meta-gamer in you knows I'm about to tell you the answer is yes. Let's think about a very common pattern for sharing memory in multithreaded code:

## Single Ownership

In most multithreaded code, memory is owned by a single thread.

That's obvious when you have threads working independently, each with its own private memory; less obviously, this is often true even of memory that gets shared between threads! The single most common way to share memory between threads is to have them take turns, so that the memory is owned by one thread *at a time*.

That's how locks work, right?

```c++
static foo an_entry;
static std::mutex lock;

void producer() {
  lock.lock();
  // (stuff)
  an_entry.ready = true;
  lock.unlock();
}

void consumer() {
  lock.lock();
  if (an_entry.ready) {
    // (stuff)
  }
  lock.unlock();
}
```

Locks are a perfectly valid way for threads to hand off ownership of shared memory. A thread tries to obtain ownership of the memory by grabbing a lock; once it has the lock, it can be certain nobody else is using the memory. When the thread is done, it hands off ownership by releasing the lock. At any given point in time, the memory either has one owner (the thread holding the lock) or no owner (when nobody has the lock).

But hey, it's not just locks that can broker temporary ownership of memory. What about our toy queue?

```c++
static toy_queue queue;
static foo an_entry;

void producer() {
  // (stuff)
  queue.put(&an_entry);
}

void consumer() {
  void *entry = nullptr;
  bool exists = queue.poll(&entry);
  if (exists) {
    // (stuff)
  }
}
```

Looks familiar, huh? 🙂 Look how the queue accomplishes the same basic ownership handoff as the lock. The producer starts off with implicit ownership of `an_entry`, and releases it by putting the entry in the queue. Later, the consumer dequeues the entry, thus obtaining ownership of the memory. In between, while the item is still in the queue, nobody owns the entry's memory.

Abstracting, both of these seem to be doing some kind of "two-phase handoff" pattern. A thread which owns the memory does something to notify the system it no longer owns the shared memory; later, a thread comes by and picks up ownership of the memory. When you put it in such a generic way, it seems likely this handoff scheme is useful in other situations too, don't you think?

But hey, notice how we've started to use words like "acquire" and "release" and "ownership." We're getting close to the summit of our long climb! How's the view looking from up here?

## Acquire and Release Mechanics

Okay, since we're already thinking about this asynchronous handoff process at a very abstract level, let's keep going. Abstractly, what can we say about *all* algorithms where one thread releases ownership of shared memory, and another thread asynchronously acquires ownership later?

First of all, we need to serialize parallel attempts to start handoff on one thread and complete it on another. So there should be exactly one memory operation which releases ownership of the memory, and exactly one which acquires it. Let's call these the **release** and **acquire** operations, respectively. So the release operation is the one that atomically begins handoff in the thread that previously owned the memory, and the acquire operation is the one that atomically completes handoff in the new owning thread later on. In between the release and the acquire, the shared memory is in that transient "no-owner" state.

What do these look like in practice? For the queue, the release is when we bump the tail pointer; after that, the entry isn't ours since it can be picked up by the consumer at any time. The acquire is when the consumer verifies the tail has been updated; that's how we know the memory is ours now:

```c++
void put(void *entry) {
  unsigned i = tail % size;
  entries[i] = entry;
  fence();
  tail++; // <-- release
}

bool poll(void **entry) {
  if (tail > head) { // <-- acquire
    fence();
    unsigned i = head % size;
    *entry = entries[i];
    head++;
    return true;
  } else {
    return false;
  }
}  
```

For a lock, the release might be a plain memory write that clears the locked bit, and the acquire might be a compare-and-swap that sets the lock bit if it's not already set:

```c++
class toy_lock {
  int state;
  
public:
  toy_lock() : state(0) { }
  
  void unlock() {
    state = 0; // release
  }

  bool try_lock() {
    return cas(&state, 0, 1); // acquire
  }
};
```

Can we say anything else about the acquire and release operations themselves? We know they're memory operations, such as a plain read, a plain write, or an atomic read-modify-write. But can any of these kinds of memory operations serve as a release or an acquire?

Well, for one, there's no bound on how long any one thread will need the shared memory, but we do want to be able to hand it off as soon as that thread's done. Really, the only way to do that is to have the releasing thread actively *do something* to indicate it's done; to complete handoff, the acquiring thread has to observe that 'something' was done.

Since a thread needs to write *something*  to indicate it's done, there's no way that a release can be a plain read; it must always involve a write. As such, we sometimes call that write a **write-release**. And since other threads need to observe the write-release, we know the acquire can't be a blind write and must always involve some kind of read, so we call that a **read-acquire**.

In practice, the write-release is almost always a plain memory write, but there are two common patterns for a read-acquire: it can be a plain memory read if the thread is simply observing the write-release handed off ownership to this thread specifically, like in the queue example. Or, the read-acquire can be an atomic read-modify-write by which the thread tries to claim the memory for itself, like with the lock.

Putting in all together, the basic scheme is:

1. Thread $A$ uses the shared memory
2. Thread $A$ does a **write-release**
3. Thread $B$ does a **read-acquire**
4. Thread $B$ uses the shared memory

I'd like to point out how *very* similar this list of steps is to another list of steps from back when we were talking about ordering and fences:

1. The entry is written to `entries[i]`
2. The entry is made 'visible' by incrementing `tail`
3. On poll, a consumer sees `tail` was incremented
4. The entry is read from `entries[i]`

That's right, it's the same basic pattern as the queue, which means it's susceptible to the same memory ordering problems we just fixed in our queue. We need to establish ordering guarantees between the write-release, the read-acquire, and the surrounding code.

What are the minimal guarantees we need? Remember, our ultimate goal is to come up with as weak a guarantee as possible, so the CPU has maximum flexibility to run our code the fastest way it knows how.

## Acquire and Release Semantics

To come up with the ordering guarantees we need, we'll think a little differently than we did before. The last time we were looking at reordering problems, we were laser-focused on the internal workings of our queue; this time, we're also concerned about what the calling thread is doing to the shared memory before and after handoff:

```c++
static toy_queue queue;
static foo an_entry;

void producer() {
  // (stuff)
  queue.put(&an_entry);
}

void consumer() {
  void *entry = nullptr;
  bool exists = queue.poll(&entry);
  if (exists) {
    // (stuff)
  }
}
```

We want to make sure all the producer's (stuff) is all done and handed off by the time the consumer starts doing (stuff). We don't want callers to handle fences and acquire-release and all that junk. In other words, we want to provide ordering guarantees for the *caller's code* that surrounds ours, in addition to making sure our code also works internally.

If you think about it, the `fence()` calls we added before actually help us here too. The put fence waits not only for our write to `entries[i]`, but also for everything the caller did before calling our `put()` method; the poll fence holds not only our read of `entries[i]`, but also everything the caller does after `poll()` returns.

But can we do better than a full `fence()`?

Well, remember that a fence is bidirectional. A `fence()` means "nothing after this starts until everything before this is finished." But isn't bidirectionality more than we need? For a write-release, we only care that stuff *before* the write has completed; it's totally fine if the CPU works ahead on something else! Similarly, for a read-acquire, we only care that stuff *after* the read waits; it's fine if stuff *before* that gets delayed.

So, instead of two full fences, how about two half fences? Let's try these rules on for size:

* A **write-release** guarantees that all **preceding** code completes **before** the write-release itself
* A **read-acquire** guarantees that all **following** code starts **after** the read-acquire itself

These two rules are called **acquire and release semantics**. In the case where your code is handing off ownership of memory across threads, this can be significantly cheaper than using full fences, because it constrains half as much code as our `fence()`-based code. And remember, the more freedom a CPU has to reorder the code, the faster it runs!

Here's an example of the full queue, now with acquire and release semantics. I've also now switched over to the full [stdatomic](https://en.cppreference.com/w/c/atomic) API to show what real working code might look like:

```c++
class toy_queue {
  
  unsigned head;
  atomic_uint tail;
  unsigned size;
  void **entries;
  
public:
  
  void init() {
    head = 0;
    tail.store(0);
    memset(entries, 0, size * sizeof(void *));
  }
  
  void put(void *entry) {
    
    unsigned t = tail.load(memory_order::relaxed);
    unsigned i = t % size;
    entries[i] = entry;
    
    tail.store(t + 1, memory_order::release);
  }
  
  bool poll(void **entry) {
    
    unsigned t = tail.load(memory_order::acquire);
    
    if (t > head) {
      unsigned i = head % size;
      *entry = entries[i];
		  head++;
      return true;
    } else {
      return false; // empty
    }
  }
  
};
```

Now `tail` is an `atomic<>` value, with each access tagged using one of three guarantees:

* `relaxed`: No ordering guarantees, just do this when it's convenient.
* `release`: Do a write-release: wait for all preceding operations to complete before doing this (but work ahead on stuff following this if it's convenient).
* `acquire`: Do a read-acquire; block upcoming operations until this is done (but allow preceding operations to be delayed if it's convenient).

Note how we use `release` when incrementing the tail, which is our queue's write-release; it can't happen until all preceding memory operations are fully done and flushed to memory. Our `acquire` is when we sample the value of `tail` in `poll()`; nothing afterward in the program can happen until we've done that. You should be able to construct a correctness proof nearly the same as we did with the `fence()`-based queue, but for the sake of brevity I won't include it here.

And that's it! Now you know what acquire and release semantics are, and when they might be useful.

In summary, acquire and release semantics are ordering guarantees used in the common situation where threads asynchronously hand off temporary ownership of shared memory. The thread which previously owned the memory releases ownership via a single write-release operation, which waits for all preceding memory operations (which might involve the memory being handed off) to complete. Asynchronously, a thread which would like to own the memory acquires ownership via a single read-acquire operation, which holds all upcoming memory operations (which might involve the memory being handed off) until the read-acquire itself has completed. Together, this is sufficient to synchronize the handoff process in the case where the releasing thread and acquiring thread run at the same time.

## That's it?

That's it. Well, *almost*.

Once you've satisfied you've grokked the above, there are a couple more factoids you should know:

**It isn't just the CPU that messes up your code like this; you're compiler's in on it too.** When you turn on optimizations, e.g. by passing one of the `-O` flags to gcc, the compiler will do all sorts of weird stuff to your code in the name of performance, using magic as black as what CPUs do at runtime. Luckily, you usually don't have to worry about this, because compilers also understand memory ordering semantics like acquire and release: if you tag your code with a memory ordering guarantee, both your compiler and your CPU will honor it. You don't do anything special about compile-time reordering.

The distinction between compile-time and runtime reordering becomes important when using compiler-specific tools like the C/C++ [`volatile` keyword](https://en.cppreference.com/w/cpp/language/cv), which provides guarantees around compile-time code transformations but doesn't constrain the CPU at all. Adding to the confusion, Java also has its own `volatile` keyword that means something completely different; Java's `volatile` makes reads and writes to a given variable behave a lot like our read-acquires and write-releases.

**It's not the fences you want; it's the ordering guarantees.** To make it easier to set up some of the ideas above, I presented a "fence" by describing a no-op instruction that provides memory ordering guarantees. But then later, we came up with other kinds of instructions that can also be tagged with memory ordering requirements. What's going on here?

A better way to think about this &mdash; which better matches the stdatomic API and the way some CPUs work &mdash; is to think of ordering guarantees as requirements that can be tagged on certain kinds of instructions, including plain memory reads and writes, atomic read-modify-writes, and fences. In this light, fences can be thought of as a special kind of "no-op" which supports being tagged with a memory ordering requirement.

In practice, you rarely want or need to use stdatomic fences; if there's a single atomic memory access that can be tagged, you should do that instead. [Acquire and release fences don't work the way you'd expect.](https://preshing.com/20131125/acquire-and-release-fences-dont-work-the-way-youd-expect/)

**There are even stronger ordering guarantees than anything we've talked about so far.** When we were talking about fences, we described an abstract method called `fence()`; technically, what we described was what the stdatomic library calls an acquire-release fence, and would be implemented like this:

```c++
using namespace std;
void fence() {
	atomic_thread_fence(memory_order::acq_rel);
}
```

This kind of fence serves both as a release, preventing preceding operations from being delayed past the fence, and an acquire, preventing upcoming operations from being started before the fence. Note how we're only talking about the relative order of operations on *this* CPU core; occasionally, you need to order operations happening on *different* CPU cores. For this, the stdatomic API provides sequentially consistent semantics, which you request by tagging an operation as `memory_order::seq_cst`.

Sequential consistency can be useful if you have multiple threads racing to set *different* variables, where the threads need to read each other's writes to determine who "won." A famous situation where this arises is the two `interested` flags in  [Peterson's lock algorithm](https://en.wikipedia.org/wiki/Peterson%27s_algorithm).

**Acquire and release semantics don't have any meaning on Intel- and AMD-brand processors.** On x86 CPUs, which is to say, on just about any Intel- or AMD-brand CPU you can buy right now, memory operations on a single CPU core happen in program order. In other words, these CPUs *don't* play the reordering tricks we've been worried about all along, and our original "broken" queue actually would have worked on these CPUs, without fences (so long as the compiler didn't play any tricks of its own).

This makes it easy to write buggy lock-free code that also happens to work flawlessly 🙂. If you forget to tag a variable access as an acquire or release, it may still work just fine because your CPU is always upgrading *every* read to read-acquire and write to write-release. But the first time you try to run your code on *somebody else's* CPU, you should try putting your ear against their computer's case so you can hear all the looney tunes "bang boom pow" sounds coming from their CPU as it executes your code.

Anyways, that's all I got! Hopefully that helped and I wasn't too boring. See ya next time!
