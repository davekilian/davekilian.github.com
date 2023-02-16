---
layout: post
title: Making Sense of Acquire-Release Semantics
author: Dave
---

*Multiprocessor Synchronization* was one of my favorite classes during my undergrad &mdash; it had a clear progression from theory to practice, starting at the theory of consensus numbers and moving onto atomic operations and then synchronization primitives and lock-free data structures, all from first principles. With careful thought and a little intuition-bending, every problem could be stated clearly and solved in a neat, orderly way ... in Java.

Later on, when I tried to translate what I learned into C, I learned why the course was taught in Java. 🙂

Java does a lot more to help you write correct multithreaded code than I realized back then. I knew having a garbage collector was helpful, and I guess I sort of knew `volatile` did *something*, but I didn't appreciate the extent of what that *something* is. Without it, the algorithms and techniques I learned in my multiprocessor course still worked, but coding them correctly required a much deeper understanding of how computers run code at the most basic level.

In this post, I want to introduce, or maybe re-introduce you to *acquire-release semantics*. Plenty of developers who can competently write lock-free code in native languages like C, C++ and Rust still have a fuzzy idea of what these are or when to use them. They're actually not so hard in and of themselves &mdash; what makes them difficult is the misconceptions you may harbor about what goes on at the hardware level when your code is running. So, in a way, that's really what I want to explain in this post.

When we're done, you should be able to answer the following question confidently:

<center>When I use acquire or release semantics, <i>what</i> is being acquired and released?</center>

The short answer is *ownership of other memory*, but that probably leaves you with more questions than answers. So let's start from the beginning! It'll be a long climb, but I think you'll like the view from the top!

To start us out, read this in your best [Gilfoyle](https://www.hbo.com/silicon-valley/cast-and-crew/bertram-gilfoyle) impression:

*"If memory worked the way you thought it did, you'd never need to acquire or release anything. Unfortunately, memory doesn't work the way you think it does."*

## How you think memory works

For the sake of example, let's say you're building a lock-free FIFO queue.

There are many strategies for building a queue; we'll use a circular array. Let's say there's a single 'producer' thread solely responsible for enqueuing items, as well as another 'consumer' thread solely responsible for dequeuing those items. So, we're building a *lock-free single-producer single-consumer ring queue*. If this seems contrived, pretend we're working with Linux's [io_uring](https://en.wikipedia.org/wiki/Io_uring) interface, where a ring is two of these queues: a *send queue* that submits I/O from userland to the kernel, and a *completion queue* for the kernel to deliver results back to the user thread.

Here's a 'toy' implementation of this queue design:

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

For brevity, I skipped heap allocating the entry array, and we'll just have to hope the caller never overflows the queue. We're going to revisit this code several times throughout the post, so make sure you get it before moving on. All good? Then here's a puzzle for ya:

There's a bug in this queue, and I'm wondering if you can find it. When you run this on a real computer, sometimes `poll()` returns success without giving you an entry. That is, `poll()` returns true, but `entry` is still null &mdash; even though the producer enqueued something non-null. Here's a repro program that sometimes demonstrates the issue:

```c++
static toy_queue queue;
static foo an_entry;

void producer() {
  queue.put(&an_entry);
}

void consumer() {
  void *entry = nullptr;
  bool exists = queue.poll(&entry);
  if (exists) {
    // oops, entry is null!
  }
}
```

Since we're letting the producer and consumer threads race, there are two valid outcomes for the consumer:

1. `exists` is false and `entry` is null (poll serialized before put)
2. `exists` is true and `entry` points to `an_entry` (put serialized before poll)

Each of these things happens some of the time. But sometimes, `exists` is true, but `entry` is still null. How could *that* happen? Think it over!

If you think it's obvious this can happen, try working backwards:

* We know `poll()` returns true
* That means `poll()` must have seen `tail > head`
* Which means `put()` must have already incremented `tail`
* But incrementing `tail` is the *last* step of a `put()`
* And checking the `tail` is the *first* step of a `poll()`
* So `put()` finished before `poll()` even started!
* ... but then, where did the entry go?

Just to show you there's nothing up my sleeve, let's try combining the logic of both threads into a single log of the whole program. If we do that, the following order should be the only one possible: notice how moving the first line of `poll()` up so it happens even one line earlier, would change the return value of `poll()` to `false`:

```c++
put: unsigned i = tail % size
put: entries[i] = entry
put: tail++
poll: if (tail > head)
poll: unsigned i = head % size
poll: *entry = entries[i]
poll: head++
poll: return true
```

But this clearly has `put()` writing to `entries[i]` before `poll()` reads it. So why does `poll()` read null?

See, not so obvious! If I've managed to stump you now, then allow me to reveal what's wrong. We fell victim to one of the classic blunders, the most famous of which is "never get involved in a land war in Asia," but only slightly less well-known is this:

## Memory isn't linearizable!

Have you used a debugger before? If yes, I suspect it affected your intuition of computer memory. If you've ever stepped through code line by line, watching as your variables change each time you step, it's tempting to think that's how code really works: running one line at a time (or at least, one assembly instruction at a time), with the contents of your variables changing in memory instantly during each step.

We can say this more rigorously than "like debugging." In fact, there's a theoretical framework that captures this intuition with formal mathematics, called **linearizability**. The basic idea is that, if your computer guarantees linearizability, then you should always be able to identify a single point in time during each instruction when the memory changes for that instruction instantly became visible to every thread simultaneously. That point in time is called a *linearization point*.

If you can collapse each thread's logic into a sequence of linearization points, it's been proved formally that you can do something that intuitively seemed right all along: combining the linear histories of two threads into a single linear history that explains everything that happened in each thread. Sound familiar? Because, that's exactly what we just *failed* to do for our repro program! The reason is simple: there's no such thing as a single global order of memory operations on a real computer, memory isn't linearizable, and what you see in a debugger isn't what happens at runtime!

![](/assets/debuggers-lie.jpeg)

Take a closer look at the body of `toy_queue::put()`:

```c++
unsigned i = tail % size;
entries[i] = entry;
tail++;
```

Remember, your CPU doesn't directly read and write memory; memory is many times slower than a CPU, so the CPU reads and writes memory through a cache. Maybe when running this code, it so happens that we get a cache hit for `tail` and `size`, but we miss on `entries[i]`. Now we're blocked: we can't execute the second line of this function until the memory contents come into the cache, and memory is slow, so that might take a while. Wouldn't it be nice if the CPU could work ahead while it waits?

How about we execute the third line of code while we're blocked on the second? In other words, why not run the program *as if* it were written like this?

```c++
unsigned i = tail % size;
tail++; // <-- do this early
entries[i] = entry;
```

Sure, it's not what the code says, but if we made this change, how would anyone know the difference? Either way, `tail` gets incremented and `entries[i]` gets initialized; neither value gets read back until both have been written. If your code can't even tell I changed the order, then changing the order can't possibly break the code, right? Right! So some CPUs are designed to make changes like this to your code, speeding it up without impacting correctness.

Hm, but we *did* impact correctness, didn't we? Looking at `put()` narrowly through a single-threaded lens, it doesn't seem like changing the order of these two memory writes matters, but when we zoom out and look at the whole program, it's definitely broken now. This two-line swap introduced a race: if you bump `tail` before writing `entries[i]`, you're reporting there's an item in the queue *before* the item is actually in the queue! So now the consumer might go and read `entries[i]` before you write it!

```c++
put: unsigned i = tail % size
put: tail++               <-- oops
poll: if (tail > head)    <-- oh no
poll: unsigned i = head % size
poll: *entry = entries[i] <-- stop
poll: head++
poll: return true
put: entries[i] = entry   <-- no
```

Well, that explains it: `poll()` sometimes returns true without an entry, because that's really the state of the queue: `tail` really says there's an entry available, and `entries[i]` really hasn't been set yet. But hey, our code doesn't do that &mdash; the CPU did. This is a highly unusual situation, wouldn't you say? I mean, how many times have we all made fun of this guy?

![](/assets/cpu-is-wrong.jpeg)

Well, this time, Principal [Software Engineer] Skinner might have a point. We wrote correct code, and in trying to dynamically optimize it, the CPU transformed our correct code into broken code. But if we were to lodge this complaint with someone who designs CPUs, they might turn this around on us: *our* code is wrong because *we* were wrong to assume linearizable memory. 

![](/assets/code-is-wrong.jpeg)

This is an old argument hashed out many times on remote corners of the Internet. Regardless of your position, ultimately what matters is CPUs aren't going to change, which means our code has to.

## So what do we do now?

If you're meta-gaming, you might think this is the point we break out the atomics &mdash; add some fetch-and-adds or compare-and-swaps and our problems will disappear, right? Not quite, I'm afraid.

Atomics are useful for solving a different problem: they resolve conflicts between parallel updates to the same variable. For example, if we had multiple threads trying to increment `tail` at the same time, we could use an atomic fetch-and-add to serialize the updates so each thread does something meaningful. But there are no such conflicts in our queue. We have exactly one producer thread managing all updates to `entries` and `tail`, and one consumer thread managing `head`; those three are our only variables. And if all updates to a variable always come from the same thread, then updates are always serial, so there are no conflicts to resolve. There are never conflicting updates to any variable in our queue, so we have no need for atomics.

Our problem is **memory ordering**.

When we put an entry in the queue, we need to update *two* variables: the queue entry and the tail pointer. Order matters: we want to write the entry first and *then* increment the tail, because updating the tail is what tells the consumer an entry is ready to be polled. If the consumer tries to poll while we're updating, as long as we haven't incremented the tail yet, the consumer will pass by without 'seeing' our in-progress update yet. It's a good plan! The only reason this didn't work is we don't have the *memory ordering guarantees* we thought we did. We were completely reasonable, but ultimately wrong in thinking writing two lines of code in order would make the CPU execute them in that order. Now that we know a CPU might shuffle our code around, we need some way to tell the CPU no, you can't do that, order matters here.

We can say that using **fences**. (Fences are also known as *barriers* &mdash; but for this post, I'll use the term *fence*).

Different CPUs express the idea of a fence in different ways. As a first-order approximation, you can think of a fence as a kind of CPU instruction, but an odd one at that: these 'fence' instructions don't do anything, they just sit there, marking points in your code like mile marker signposts on a freeway. We call them "fences" because they enact boundaries. If a CPU decides it's a good idea to push an instruction down so it happens later, or pull one up so it happens sooner, but the instruction runs into a fence along the way, then *bonk!* the instruction stops moving. Instructions can't cross a fence.

To see how this works in practice, let's imagine we have a method called `fence()` that gets compiled to a CPU fence instruction like I just described. Then here's how we might use `fence()` to fix our queue:

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
      fence();
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

See how we added a new `fence()` call to `put()`? That fixes the reordering problem we've described at length. Now if the CPU gets bored waiting for `entries[i]` to be read into the cache, and tries to pull up the `tail++` line so it happens sooner, *bonk!*, the `tail++` line hits that fence and stops moving. We've forced the entry to be written before the tail is bumped. Problem solved!

But hang on ...

![](/assets/fence-in-poll.jpeg)

There was a second bug all along! 😀 Just as the producer must make sure the entry is written before making it visible (by incrementing the tail), so too must the consumer ensure the entry is visible (tail was incremented) before reading the entry itself. Without that `fence()` call in `poll()`, a CPU would be allowed in principle to change `poll()` so the entry is read *before* the tail is checked, kind of like this:

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

Once again, this creates a race condition where the polling thread can return true, but still read null:

```c++
poll: unsigned i = head % size;
poll: void *temp = entries[i];
put: unsigned i = tail % size
put: entries[i] = entry
put: tail++
poll: if (tail > head)
poll: *entry = temp;
poll: head++;
poll: return true;
```

Notice how `temp` will be null, because `put()` really hadn't started yet; but the return value will be `true` because, by the time we checked the tail, `put()` had already completed. This is a new bug with the same symptoms: the return value is true but the entry isn't initialized. That's what the new fence in `poll()` fixes.

Now I don't know about you, but I find this second bug concerning. So far the CPU has come up with two different ways to break our code, and both were the kinds of subtle, impossible-to-repro problems that can slip through review and testing and then run roughshod through production. Even worse, the second bug had the exact same symptoms as the first one; imagine finally finding and fixing the first bug, only to find the same thing is still happening in prod days after the rollout! When is this going to end? How do we know when the CPU is out of tricks? Can we ever declare our queue correct?

Maybe we should try to *prove* our queue works now.

## Reasoning about Fences

No more shooting from the hip and reasoning based on what we expect. To show the bug is fixed, we need to understand what guarantees the CPU gives us, and use those to justify what we did. We need to channel the energy of that rules-lawyering friend who you keep inviting to your DnD sessions for some reason.

There are four steps needed to hand off an entry through the queue:

1. The entry is written to `entries[i]`
2. The entry is made 'visible' by incrementing `tail`
3. On poll, a consumer sees `tail` was incremented
4. The entry is read from `entries[i]`

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

If all four of these steps run in order, there is no 'missing entry' bug. There can't be: `poll()` only thinks an entry is available if it checks the tail (3) after it was incremented (2), and if everything happens in order, then the entry is written (1) before the handshake in (2-3), and is later read (4) after the handshake. So if we see an entry is available, then the entry's read always comes after the write.

The question is, do we really know the order always stays 1-2-3-4 now? Let's try checking each pairwise relationship between any two neighboring steps:

**The entry is written (1) before the tail is updated (2)**: This order is guaranteed by the fence that appears between these two steps. The fence works because both steps are in the same thread.

**The producer increments the tail (2) before the consumer sees it (3)**: This order is guaranteed by the 'obvious' rule that memory only changes if someone writes it; in this program, only the producer writes to tail. So if the consumer sees the tail was incremented in (3), then the producer must have previously incremented it in (2).

Note we have no guarantee how long it takes after the producer writes the tail (2) for the value to become available for the consumer to read (3); however, this doesn't affect the relative order of (2) and (3), so for this proof, the delay is not important.

**The consumer sees the new tail (3) before it reads the entry (4)**: Once again, we have two operations in the same thread that appear on opposite sides of a fence, so they happen in program order.

And there you go. Even though the CPU might start step 1 way earlier than you might expect, and even though step 4 might start way late, and even if plenty of time could pass between steps 2 and 3, none of the matters; we've established that the relative order of all four steps will always be 1-2-3-4 using rules backed by guarantees from the CPU's designers. We have finally rid ourselves of that bug! 🎉

Our queue now works, so we *could* declare ourselves done ... but you want to go the extra step, right? Because, the `fence()` method we came up with here works, but it isn't particularly efficient. And besides, we don't even know how to implement the `fence()` method yet. But for now, let's focus on the performance aspect:

Consider this: when you have a bunch of bugs assigned to you, and you have a fix in code review, what do you do? I mean, if the answer is "sit around and wait until my code is merged," yo, I don't judge; but I think many people would start working on the next bug in the meantime. If that's you, then you understand the value of being able to take work items, break them down into smaller pieces, and then interleave those smaller pieces of work to stay busy so you get things done sooner. That's pretty much what a CPU is doing when it "works ahead" in the way that was breaking our multithreaded code. The more you limit a CPU's ability to break down the work and shuffle the pieces around, the more time it has to spend waiting instead of getting things done.

In that light, `fence()` *might* be overkill. Can we do something else, something with weaker (meaning more flexible) memory ordering guarantees?

Well, it's tricky. There are a lot of things you can do in hardware *in principle*, but you only have space for so many transistors, which means anything you're going to put in hardware had better be for something a lot of software can use. Whatever we invent here has to work for lots of multithreaded programs, not just our queue. Luckily, our queue happens to use a *very* common pattern we can certainly optimize for:

## Single Ownership

In most multithreaded code, memory is owned by a single thread.

That's obvious when you have threads working independently, each with its own private memory. But even when threads share memory, you almost always set it up so the threads *take turns* with it. In this setup, you could claim the memory has a single owner, because at any given point in time, you can identify a single thread that owns the memory: namely, whichever thread is taking its turn right now. (By the way, this also happens to be the principle underlying Rust's `RefCell<>` type.)

One-at-a-time ownership ... that's how locks work, isn't it?

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

Locks are a perfectly valid way for threads to hand off ownership of shared memory. A thread tries to obtain ownership of the memory by grabbing a lock; once it has the lock, it can be certain nobody else is using the memory; this thread is the sole owner. When the thread is done, it hands off ownership by releasing the lock. At any given point in time, the memory either has one owner (the thread holding the lock) or no owner (when nobody has the lock).

But hey, it's not just locks that can broker ownership of memory. What about our queue?

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

Look how the queue accomplishes the same basic ownership handoff as the lock. The producer starts off with implicit ownership of `an_entry`, and releases it by putting the entry in the queue. Later, the consumer dequeues the entry, thus obtaining ownership of the memory. In between, while the item is still in the queue, nobody owns the entry's memory.

Looking at both of these examples and abstracting, it seems like we have stumbled upon some kind of "two-phase handoff" pattern. A thread which owns the memory does something to release it, notifying everybody it is no longer the current owner; later, a thread comes by and acquires ownership of the memory. When you put it in such a generic way, it seems likely this handoff scheme is useful in a wide variety of situations, not just the few we've already talked about, don't you think? Let's try to optimize for this situation.

By the way, look how we've started to use words like "acquire" and "release" and "ownership." We're getting close to the summit of our long climb! How's the view looking from up here?

## Acquire and Release Mechanics

Okay, since we're already thinking about this asynchronous handoff process at a very abstract level, let's keep going. Abstractly, what can we say about *all* algorithms where one thread releases ownership of shared memory, and another thread asynchronously acquires ownership of said memory later?

First of all, it's always possible for one thread to start handoff in parallel with another thread checking if it can complete handoff; these two steps have to be serialized. That means there should be exactly one memory operation to acquire ownership, and exactly one to release it. Let's call these the **acquire** and **release** operations, respectively. Since the acquire and release operations must be serialized, they need to operate on the same variable, and they need to both be atomic; that's what allows us to serialize them relative to one another. Note that a plain memory read or write counts as "atomic" for our purposes here, as do the read-modify-write operations like fetch-and-add or compare-exchange we normally think of when we say "atomic."

If this sounds confusing, it might help to see these two operations in more concrete terms.

For the queue, the variable that both threads synchronize on is `tail`. The release is when the producer bumps the tail pointer; after that, the entry is no longer producer-owned since it can be picked up by the consumer at any time. The acquire is when the consumer later verifies the tail has been updated; that's how the consumer knows it now owns the memory:

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

For a lock, we typically use a `state` variable that has a single bit indicating whether someone currently has the lock. That's the variable threads synchronize on; the release might be a plain memory write that clears the locked bit, and the acquire might be a compare-and-swap that sets the lock bit if it's not already set:

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

Does that kind of make sense? If so, then let's go back to thinking abstractly. What else can we say about the acquire and release operations in *all* possible algorithms that hand off ownership like this?

Here's an interesting observation: there's no bound on how long any one thread will need to own the shared memory, but we want to be able to hand off the memory as soon as a thread is done with it. Since there's no timing guarantee, the only way to make this work is for the releasing thread to *actively* notify other threads that it's done now, which means it has to write something to memory. So a release must involve some kind of memory write. Similarly, other threads must observe that write, so an acquire must involve some sort of read.

In practice, a release is often just a plain memory write. There are two common patterns for the acquire: it can be a plain memory read, if it's clear which thread owns the memory next, like with the queue. Or, the acquire can be an atomic read-modify-write by which the thread attempts to claim ownership for itself, like with the lock.

Putting in all together, the basic scheme is, abstractly:

1. Thread $A$ owns the shared memory and does stuff with it
2. Thread $A$ does something involving a **write** to **release** ownership
3. Thread $B$ does something involving a **read** to **acquire** ownership
4. Thread $B$ owns the shared memory and does stuff with it

Just like we saw with the queue earlier, though, we can't implement sequential protocols like this without establishing ordering guarantees! So what guarantees do we need here? Remember, we're trying to come up with the weakest (most flexible) guarantees we can. Let's think about that next:

## Acquire and Release Semantics

We're back to thinking about fence-like things and ordering guarantees, but this time, our goal is subtly different from before. Back then, we were laser-focused on the internal workings of our queue; this time, we're also concerned about what the calling code is doing to the shared memory before and after we hand off ownership. To see what I mean, consider this queue program again:

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

We want to make sure all the producer's (stuff) is done by the time the consumer starts doing its (stuff). We don't want bog down callers handling fences and acquire-release and all that junk; we want to provide ordering guarantees for the *caller's code* that surrounds ours, in addition to making sure our code also works internally.

Interestingly, in our current queue implementation, we don't need to do anything special to support this. The `fence()` method has a global effect on order: the one in `put()` separates everything that came before (including the calling code and our write to the entry array)  from the point we update the tail and everything thereafter. Similarly, the one in `poll()` separates everything that came before (including when we check the tail index) from everything that comes afterward, including when we read the entry from the entry array *and* everything the caller does thereafter).

But we don't want a full `fence()` here; can we come up with something more flexible? Remember, the more freedom a CPU has to reorder the code, the faster it runs!

Well, here's an idea: `fence()` is bidirectional. It means "finish everything that came before this before starting anything that comes after this." But isn't bidirectionality more than we need? For a release, we only care that stuff *before* the release has completed; it's totally fine if the CPU works ahead on something else! For an acquire, we only need the acquire to block stuff that comes *afterward*; it's fine if stuff *before* gets delayed!

So, instead of two full fences, how about two half fences? Let's try these two rules on for size:

* A **write-release** guarantees that all **preceding** code completes **before** the releasing write
* A **read-acquire** guarantees that all **following** code starts **after** the acquiring read

Together, these two rules are called **acquire and release semantics**. They work a little differently than fences we've been describing so far. Instead of standalone CPU instructions, we think of acquire and release semantics as requirements you can attach to a memory operation. When you're releasing ownership of memory by writing to some variable, you can tag that write as needing *write-release semantics* to ensure all preceding code finishes before the write finally happens in memory. Similarly, if you're trying to acquire ownership of memory by doing something that involves a read, you can tag that read as needing *read-acquire semantics* to ensure the code that follows the read does not start until you have actually read the value from memory.

That's ... really about all there is to it! Now you know what acquire and release semantics are! Not so bad once you get over your old ideas about memory, eh?

Here's a final example implementation of our old friend, the lock-free single-producer single-consumer ring queue, now using acquire and release semantics. I've also switched over to the full C++ [stdatomic](https://en.cppreference.com/w/c/atomic) API to show what 'tagging' memory operations with ordering semantics looks like in practice:

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
    unsigned tl = tail.load(memory_order::relaxed);
    unsigned i = tl % size;
    entries[i] = entry;
    tail.store(tl + 1, memory_order::release);
  }
  
  bool poll(void **entry) {
    unsigned tl = tail.load(memory_order::acquire);
    if (tl > head) {
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

I've made `tail` an `atomic<>` type, and tagged each access with one of three semantics:

* `acquire`: This is a read-acquire; block upcoming operations until this is done (but allow preceding operations to be delayed if it's convenient).
* `release`: This is a write-release: wait for all preceding operations to complete before doing this (but work ahead on stuff following this if it's convenient).
* `relaxed`: No ordering guarantees, do this any time it's convenient.

We use the `release` semantic when bumping the tail, since that's how the current thread releases its ownership of the item it just enqueued. Past this point, the consumer can pick up the item off the queue at any time, so we need to be certain this thread isn't still running code that might be using the shared memory, in this method or in the calling context.

Similarly, we use the `acquire` semantic when reading the tail in `poll()`, since that's how we know ownership of an entry has passed to us. We need to be certain this thread isn't already running code on the shared memory until we've observed the producer's releasing write.

Finally, I should note the `relaxed` read in `put()` cannot cross the write-release at the end of that method. Acquire-release semantics impose order on relaxed memory operations the same as they do for regular variable reads and writes. In fact, you can think of 'normal' variable reads and writes as 'having relaxed semantics.'

And there you have it ... memory order, ownership, and acquire and release semantics!

## Parting Notes

Here are a few more things you might want to know. You can consider these areas you might want to do some further research to broaden and deepen your knowledge of what we talked about today:

**It isn't just the CPU that messes with your code; you're compiler's in on it too.** When you turn on optimizations, e.g. by passing one of the `-O` flags to gcc, the compiler will do all sorts of *fun* stuff to your code in the name of performance, using magic as black as what CPUs do at runtime. Luckily, you usually don't have to worry about this, because compilers also understand memory ordering semantics like acquire and release: if you tag your code with a memory ordering guarantee, both your compiler and your CPU will honor it. You usually don't do anything special about compile-time reordering.

**The `fence()` method is kind of an imaginary construction.** Using the stdatomic library, the closest you can get to the fence semantics we described in this post is to use an "acquire-release" fence, meaning it has both the semantics of an acquire (preventing upcoming operations from being started before the fence) and a release (preventing preceding operations from being delayed past the fence). You might implement `fence()` like this:

```c++
using namespace std;
void fence() {
  atomic_thread_fence(memory_order::acq_rel);
}
```

However, this is only an approximation of what we were talking about; it actually does something [more subtle](https://preshing.com/20131125/acquire-and-release-fences-dont-work-the-way-youd-expect/ ) than what I described. In general, use the `atomic_thread_fence` method with caution, as it's unintuitive.

**There are even stronger ordering guarantees than anything we've talked about so far.** The acquire and release semantics we've discussed in this post *only* affect the relative order instructions take effect in memory. They *don't* require the instructions to take effect right away! So for example, if you tag a write as a write-release, then you guarantee all pending memory writes in your program *will eventually* complete before the releasing write *eventually* completes, but there's no guarantee that any of this happens before the CPU moves on to the next instruction. All of your writes could still be [buffered inside your CPU](https://en.wikipedia.org/wiki/Write_buffer)!

Occasionally you need a guarantee that your writes really are in memory before the next line of code executes. A famous example of this involves the two 'interested' flags in  [Peterson's lock algorithm](https://en.wikipedia.org/wiki/Peterson%27s_algorithm). To support code that needs these kinds of immediacy guarantees, the stdatomic API offers *sequentially consistent* semantics, which can be tagged using `seq_cst`. As you might have guessed, `seq_cst` is even slower than `acq_rel`.

Sequential consistency is about the closest you'll ever get to linearizable memory in practice. Remember how, at the start of this post, I mentioned underappreciating the heavy lifting Java's `volatile` keyword had been doing for me? [Java's `volatile` keyword offers sequential consistency](https://bartoszmilewski.com/2008/11/11/who-ordered-sequential-consistency/) for all reads and writes to all variables you mark volatile. You will have a hard time finding any of this acquire or release stuff in a Java program!

**Acquire and release semantics don't have any meaning on Intel- and AMD-brand processors.** On x86 CPUs, which is to say, on just about any Intel- or AMD-brand CPU you can buy right now, memory operations on a single CPU core happen in program order. These CPUs *don't* play the reordering tricks we've been worried about all along, and our original "broken" queue actually would have worked on these CPUs without fences or acquire-release semantics or anything (so long as the compiler didn't play any tricks of its own). However, these CPUs do buffer stores, so they're not sequentially consistent unless you use fence instructions such as `MFENCE`.

This puts us in a kind of funny situation: according to the stdatomic docs, and the C++ spec, our original queue is technically wrong, but on some pretty popular CPUs, it'll work just fine anyways. But the first time you try to run your code on a different kind of CPU, well, you should try putting your ear against their computer's case so you can hear all the looney tunes "bang boom pow" sounds coming from their CPU as it executes your code. It's a good idea not to assume code that uses acquire/release semantics is correct unless you've tried running it on a CPU architecture that does more aggressive reordering, such as ARM or PowerPC.

With that, in terms of acquire and release semantics ...

![](/assets/thats-about-it.jpeg)

Hopefully that was helpful and I wasn't too boring. See ya next time!