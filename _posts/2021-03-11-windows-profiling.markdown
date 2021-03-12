---
layout: post
title: Profiling on Windows
author: Dave
draft: true
---

Some basic ideas from an email I had, probably needs restructuring to flow nicely

> The unofficial guide, based on what I know so far. Disclaimer: I do not work on ETW and have no special knowledge. Don’t trust anything other than what you read in the official docs. Just synthesizing my own notes
>
> Profiling on windows is very ‘build your own solution.’ You get a bunch of little pieces and have to stitch them together. What you're doing might not make a lot of sense until you understand a lot of little pieces - it once you understand them, the profiler can indeed be very useful.
>
> ETW: A basic system for shipping ‘events’
>
> Events have an ID, name, variable number of properties
>
> Binary logging format
>
> For compactness, no schema in the event. Need a schema to interpret. “Manifests.” Tlg schemas.
>
> ETW provides routines for tracing events. You call these as an event provider. If nobody is listening, the trace call is a cheap no-op. Otherwise it’s a low overhead memory copy into a buffer
>
> ETW provides consumer infrastructure. ETW can periodically flush buffers to a log file for reuse. You can get a callback but with caveats.
>
> One annoying thing is parameter tuning for buffer sizes and flushing. Get it wrong and ETW trace calls may encounter a full buffer; if that happens, they drop the event instead of logging it l. So, ETW is lossy.
>
> Those log files are key. WPA is a tool for visualizing data in those log files.
>
> WPA is generally not all that pretty and not all that intelligent. It doesn’t try very much to interpret the events you feed into it; it focuses more on aggregating, sorting and filtering events in a big old table.
>
> One last aside: one thing consumers can do is request ‘stack walking.’ If the consumer enables this, the producer’s “trace event” routine will capture a backtrace of the current thread into the event.
>
> Okay, so ETW is just a generic machine local pub/sub system for these small binary ‘events.’ We need to know about it because it is the vehicle for collecting profiler information
>
> To facilitate profiling, the NT kernel has a few well known EventTrace calls nestled in some strategic locations.
>
> One is the ‘profiler’ event. When someone starts listening for this event, the NT kernel configures each processor to interrupt a lot (maybe a million) times per second. Each time that interrupt fires the interrupt handler calls EventTrace and then returns to whatever was happening before 
>
> So you start listening for this event and turn on stack walking. Now every microsecond or so, your processor stops what it’s doing and captures a backtrace of whatever the program was doing. This gets shipped to an ETW buffer that later gets flushed to an ETL file. You plug that into WPA and whammo - you’ve got something that looks a lot like a sampling profiler!
>
> Sampling profilers are good at telling you what your processor is doing when it’s doing stuff. Although it’s capturing backtraces at random times, LLN says over time this will give you a complete and accurate picture of what’s going on system-wide. This works particularly well if you’re executing the same code over and over, such as the main logic for a video game or the request processing logic for a server under load.
>
> One tricky thing with sampling profilers is defining what it means for code to be ‘running.’ Modern processors are a lot faster than memory, and when you execute an instruction that has to access memory, that instruction ‘pauses’ the CPU until the memory operation completes. The sampling profiler only cares what’s running, not why, so memory accesses show up 
>
> Another thing to keep in mind that sampling profilers are showing you a breakdown of CPU time that, ultimately, sums to 100%. It can’t tell you what’s faster or slower than it should be; it can only show a breakdown of CPU time. This is incredibly important for servers because if you fix a bottleneck and trace again, if you are now able to push *more* traffic than before, the traces are no longer apples-to-apples and easily comparable.
>
> However, this view can be valuable for finding code that’s more expensive than you expect.
>
> One thing sampling profilers are not good at is telling you that your processor isn’t running, or why. For that, you’ll want a different well-known NT ETW event: CSWITCH. This event is traced every time the kernel starts running a thread on a processor. Like the profiler event, you can turn on stack walking for the CSWITCH event to see what code your thread is starting up in.
>
> Why is this useful? Waits. If you’re trying to debug why code is slow, a profiler won’t help you if your code is not running. This might happen because you’re waiting for a lock, or you’re calling some other Windows routine that blocks until something happens.
>
> One tricky thing with this is the CSWITCH event gets traced whenever an interrupt returns. In many cases you’re only interested in waits that your code initiated, rather than sporadic interrupts that were forced upon you.
>
> How would you use these two things to triage a pause?
>
> Let’s say you know a piece of code is taking longer than it should. You might start by taking a CSWITCH trace of the repro and seeing if the thread was even running during the pause; if it wasn’t, it’ll hopefully be clear why not!
>
> If you don’t see context switches, then you can be pretty certain your code was running. Nows a good time to do a profiler trace to see where those CPU cycles are going!
>
> There’s no silver bullet here; only a question of whether the breakdown matches your expectation. Often times the bug won’t be visible in the profiler trace, but the profiler can help you rule out things that are not taking a long time and focus your search in places you’re more likely to find what’s wrong.
>
> Maybe example on using WPA
>
> Maybe example xperf command line. Talk about PROC_THREAD and LOADER and what information they add that is critical for WPA to actually be able to use stack walk data
>
> Protip: don’t turn on both PROFILE and CSWITCH in the same trace. Your profiler will pick up the overhead from tracing CSWITCH stacks and all your numbers will get thrown way off.