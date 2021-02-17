---
layout: post
title: Async Fanout
author: Dave
---

This post will be short but sweet. I wanted to write down a cute coding trick I picked up from a coworker a while ago for handling asynchronous completion for operations that "fan out" to multiple sub-operations. I'll use C for the examples (if you need async I/O there's a good chance you're using C or a derivate anyways), but if you prefer some other language, the same technique probably works in that language too.

As an example, say you had a cluster of machines which are each logging diagnostics to local disks, and you'd like to stand up a 'distributed grep' service which remotely greps these logs, combines the results (maybe sorting them chronologically?), and returns those to the caller.

Let's say you already have a 'remote grep' library that looks something like this:

```c
// Log lines returned by a grep on some log files
typedef struct { ... } grep_result;

// A callback for asynchronous I/O completion
typedef void grep_callback(errno_t err, void *context);

// Remotely runs `grep <filter>` on the given machine's logs.
// May complete asynchronously by returning -EINPROGRESS
// and asynchronously calling `callback` on completion.
errno_t grep_one_host(
        const char *filter, // the query to run
        const char *host, // host to run the query on
        grep_result **result, // receives the result
        grep_callback *callback, // called on completion
        void *callback_context) // passed as `context` on completion

// Returns a grep_result consisting of all input results combined
// and then sorted in chronological order
grep_result *grep_combine_and_sort(grep_result **inputs, int count);
```

We'll use this library to implement a `distributed_grep` routine that does a `grep_one_host` on every machine in the cluster. Once the results are in, we'll `grep_combine_and_sort` them together to obtain the final output which we return to the caller.

Our function might have a signature like this:

```c
// Remotely runs `grep <filter>` on all given remote `hosts`.
// May complete asynchronously by returning -EINPROGRESS
// and asynchronously calling `callback` on completion.
errno_t distributed_grep(
        const char *filter, // the query to run
        const char **hosts, // list of hostnames to run on
        int num_hosts, // number of hosts in hostname array
        grep_result **result, // receives the result
        grep_callback *callback, // called on completion
        void *callback_context); // passed as `context` on completion
```

A straightforward way to write this routine is to loop over each hostname in `hosts`, calling `grep_one_host` on each. We'll allocate a context structure on the heap so we have somewhere to store all the intermediate results as they're coming in. Finally, when the last `grep_one_host` callback is invoked, we'll finish by calling `grep_combine_and_sort` to obtain the final result set, then invoke the caller callback to notify the caller their operation completed.

How do we know which `grep_one_host` callback is the last one? A simple approach is to store a counter in the context structure tracking how many `grep_one_host` callbacks still remain. This counter is managed using atomic instructions (e.g. [\_\_sync\_ builtins](http://gcc.gnu.org/onlinedocs/gcc-4.4.3/gcc/Atomic-Builtins.html#Atomic-Builtins) on GCC/clang or Windows's [Interlocked](https://docs.microsoft.com/en-us/windows/win32/sync/interlocked-variable-access) intrinsics). Each `grep_one_host` callback ends by atomically decrementing the counter, and whoever drops the counter to zero computes the final set and returns it.

This is all fairly standard stuff so far. Here's the cute trick: if you're reference counting async completions anyways, go ahead and have the issuing thread (the one running `distributed_grep` in our example) also hold its own reference count while it's running. That way you don't have to worry about the operation getting completed out from underneath you before you explicitly say it can.

To illustrate what that means, here's a full implementation of `distributed_grep` using the approach we described, including the reference-counting trick:

```c
#include <remote_grep.h>

#if DISTRIBUTED_GREP_ON_WINDOWS
#define atomic_incr(ptr) InterlockedIncrement(ptr)
#define atomic_decr(ptr) InterlockedDecrement(ptr)
#else
#define atomic_incr(ptr) __sync_add_and_fetch(ptr, 1)
#define atomic_decr(ptr) __sync_sub_and_fetch(ptr, 1)
#endif

typedef struct {
    grep_callback *caller_callback; // invoked when we're done
    void *caller_callback_context; // the callback's context parameter
    grep_result **caller_result; // receives the final combined result set
    grep_result **intermediate_results; // array of per-machine results
    int num_results; // number of items in intermediate_results array
    volatile errno_t err; // overall result of the operation
    volatile int refcount; // number of completion references remaining
} distributed_grep_context;

errno_t distributed_grep(
        const char *filter,
        const char **hosts,
        int num_hosts,
        grep_result **result,
        grep_callback *callback,
        void *callback_context)
{
    distributed_grep_context *context = malloc(...);
    context->caller_callback = callback;
    context->caller_callback_context = callback_context;
    context->caller_result = result;
    context->intermediate_results = malloc(...);
    context->num_results = num_hosts;
    context->err = 0;
  
    //
    // Since we're going to issue num_hosts I/Os, initiailize the
    // number of outstanding completions to num_hosts + 1; the extra
    // reference is held by this routine until we're completely
    // done setting up I/O operations.
    //
  
    context->refcount = num_hosts + 1;
  
    //
    // Issue all remote grep calls. They'll run in parallel and
    // each complete asynchronously with a call to our callback,
    // grep_one_host_callback, defined later in this listing.
    //
  
    for (int i = 0; i < num_hosts; ++i) {
    
        int err = grep_one_host(
                filter,
                hosts[i],
                &context->intermediate[i],
                &grep_one_host_callback,
                context);
      
        if (err != -EINPROGRESS) {
            grep_one_host_callback(err, context);
        }
    }
  
    //
    // Once a grep_one_host call has been accepted, it's possible for
    // it to call grep_one_host_callback on another thread at any
    // time, even whiel this routine is still running - even before
    // the grep_one_host call above returns control to this routine!
    //
    // However unlikely, it's possible that *all* grep_one_host calls
    // have already been completed asynchronously, and their callback
    // have all been called, by the time we get here.
    //
    // That's okay, though. This routine is holding its extra completion
    // reference, so we know none of the callbacks has completed the
    // overall operation out from underneath us. From the caller's
    // perspective, the operation is still outstanding, and internally,
    // the context variable is still safe to dereference.
    //
    // If there's any more work we wanted to do with the context,
    // now's the time to do it.
    //

    do_something_with_dgrep_context(context);
  
    //
    // Drop our extra completion reference now that we're about to
    // return.
    //
    // If we're in the case described above, then the reference we
    // drop will be the last one, so we'll call our completion logic
    // inline before returning.
    //
  
    if (0 == atomic_decr(&context->refcount)) {
        complete_distributed_grep(context);
    }
  
    return -EINPROGRESS;
}

static void grep_one_host_callback(errno_t err, void *callback_context)
{
    distributed_grep_context *context = callback_context;
  
    if (err != 0) {
        context->err = err;
    }
    
    if (0 == atomic_decr(&context->refcount)) {
        complete_distributed_grep(context);
    }
}

static void complete_distributed_grep(dgrep_context *context)
{
    errno_t err = context->err;
  
    if (err == 0) {
        *(context->caller_result) = grep_combine_and_sort(
                context->intermediate,
                context->num_hosts);
    }
  
    grep_callback *caller_callback = context->caller_callback;
    void *caller_context = context->caller_callback_context;
  
    cleanup_dgrep_context(context);
  
    caller_callback(err, caller_context);
}
```

The handy thing about this trick is to make it very explicit at what point it becomes possible for the `distributed_grep_context` to get completed out from underneath the `distributed_grep` routine while it's still running.

Technically, the extra reference held by the issuing thread (`distributed_grep` call, in our example) isn't always needed. As long as the last thing the issuing thread ever does with its context is issue an I/O, it doesn't need to explicitly protect the context with a completion reference. That said, having it hold the unnecessary reference anyways isn't going to add much code or significantly impact performance, and it's a nice way to make it clear to future maintainers when `context` goes 'asynchronously' out of scope and can no longer be dereferenced. (Say, for example, that someone comes along a year after this code was written and needs to add that 'do something' call.)

That extra reference was all I wanted to mention. I just think it's neat!

<img src="https://i.kym-cdn.com/entries/icons/original/000/029/906/DjoYh9xUUAA0IYv-2.jpg" alt="meme" style="zoom:25%;" />