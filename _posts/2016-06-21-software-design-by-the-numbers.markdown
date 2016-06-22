---
layout: post
title: Software Design with Math
author: Dave
draft: true
---

Developing software involves a constant barrage of decision-making.
We software developers make making these decisions way too complicated.
Why not simplify our lives with some basic math?

Here's some basic math to guide your decision-making process.

## When to Rewrite Stuff

If one thing's true in life, it's that there's never enough time.

No matter the scale, whether we're looking at a small function or a huge codebase, whether to rewrite is a question of whether rewriting finishes the job faster than not rewriting.
By finishing the job we mean the whole job, with all the bells and whistles and future maintenance.

Say if you don't rewrite, the time it'll take to do the feature is 

* (time needed to get the feature working on the existing codebase) +
* (time needed to stabilize)

If you do rewrite, you're dealing with a longer lists of time costs:

* (time needed to rewrite the thing) +
* (time needed to stabilize the thing after the rewrite) +
* (time needed to add the feature after rewriting) + 
* (time needed to stabilize new feature)

To make the decision of whether to rewrite, just sume both lists and choose whichever takes less time.

## Where to Add Infrastructure

The number of bugs in your code is linearly proportional to how much code you have.
So less code is always better, if you can manage it.

Adding infrastructure to your code can mean faster turnaround times for changes you expect you'll need to make.
But adding infrastructure also means more code and more bugs, so adding more isn't always great.
Then when is it a good idea?

One heuritic: the code that has had the most (bugs, requirement changes, feature requests) is the most likely to have more (bugs, requirement changes, feature requests) in the future.

So it's okay to start with a 'simple' draft for everything, and add infrastructure as soon as you see a recurring pattern of bugs and changes in a particular area of your codebase.

Beyond that, you just need some experience and some good judgment ;-)

## Design Speculation

Adding infrastructure is actually speculating in your software design.

You can design up front to accommodate change, but you can't accommodate every possible change.
One way or another, you're going to need to make educated guesses about what changes are likely, to accommodate for them in advance.
(That's why the 'senior' developers on your team or usually more heavily embroiled in the business and user scenarios, whereas more junior developers often stick to the code itself).

The thing about speculating is, once it comes time to make changes to the code:

* If you were right, the change will be easy
* If you were wrong, the change will be very expensive

For example, let's say if you speculate correctly, then when the time comes to make a change, it'll only take $\frac{1}{10}$th the time it would have taken had you not acommodated for the change up front.
Let's also say if you're wrong, then when the time comes to make the change, it will take 10 times as long.

Then to break even, what's the necessary probability you speculated correctly?

$p(\frac{1}{10}x) = (1 - p)(10x)$

$\frac{1}{10}p = (1 - p)(10)$

$\frac{1}{10}p = 10 - 10p$

$10.1p = 10$

$p = .99$ (roughly)

So, you want to add infrastructure; are you 99% sure you're gonna need it?

Another facet to consider: the 10X slowdown from speculating wrong is a way bigger hit than the 10X speedup from speculating right.
Using the numbers above,

* If making the change would take 1 hour had you not speculated,
* Then the change would take 6 minutes if you had speculated (you saved 54 minutes)
* And the change would take 10 hours if you had speculated wrong (you lost 9 hours!)

In light of this, experienced developers often keep it simple by default and leave adding complexity to the future.
This means future changes aren't free, but they usually don't require expensive reworks either.

Over the long run, how often do you have to speculate correctly to break even with never speculating at all?

$1x = (\frac{1}{10}x)p + (10x)(1 - p)$

$1 = (\frac{1}{10})p + (10)(1 - p)$

$1 = (\frac{1}{10})p + 10 - 10p$

$9 = 9.9p$

$p = .90$ (roughly)

So, do you speculate correctly 90% of the time?

Keep it simple and avoid speculating if you can -- because the odds are against you!

