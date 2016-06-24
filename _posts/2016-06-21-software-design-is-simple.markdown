---
layout: post
title: Software Design is Simple
author: Dave
draft: true
---

There's lots of great advice out there about how to design software, and it's all just a few keystrokes away.
The only problem with all this information is there's way, way too much of it.
Spend a few hours reading all that advice, and you might get the impression that designing software is complicated!

Let's be clear -- designing software may be hard at times, but in the end it's not really all that complicated.
The end goal is always the same:

* Write software that solves a problem,
* does exactly what you mean it to,
* and takes as little time to develop as possible

There are lots of details to think through, but if you remember the goal, the details tend to fall in line.
Then all that's left is to learn the problem space and think it through!

Now, watch as we use our goal statement to reduce all those hair-pulling, information-overload decisions into a little rote research and one simple, unglamorous informed decision.

## When to Rewrite Stuff

Typical guidance for when to rewrite is over-complicated.
Remember our end goal?

> * Write software that solves a problem,
> * does exactly what you mean it to,
> * and takes as little time to develop as possible

For now, let's take for granted that rewriting is a purely technical decision, that it doesn't affect your end product's interface or feature set.
Thus we can safely say rewriting doesn't affect whether our software solves the problem at hand, because rewrite or no, from the user's point of view the software ought to work the same.

That leaves us with variables to minimize:

* How long it takes to write it and fix bugs we introduced
* The number of bugs still around when we decide to ship

A tall order, no doubt.
But we can simplify the decision.
Say if you don't rewrite, the time it'll take to do new features is 

* (time needed to get the feature working on the existing codebase) +
* (time needed to fix bugs from those changes)

If you do rewrite, you're dealing with a longer lists of time costs:

* (time needed to rewrite the thing) +
* (time needed to fix bugs from those changes)
* (time needed to add the feature after rewriting) + 
* (time needed to fix bugs from those changes)

To make the decision of whether to rewrite, just sum both lists and choose whichever is less total time.
If there are additional time costs to consider, just expand the lists to account for everything.

## Where to Write Fancy Code

---

Add here what was previously the design speculation section

In the end, after declaring just how sure you need to be that fancy code will be worth it, provide a heuristic:

> The code that has had the most (bugs, requirement changes, feature requests) is the most likely to have more (bugs, requirement changes, feature requests) in the future.

But by default, the answer really is fancy code is never excusable until it's proven necessary.
And even then, true cleverness is recasting the problem so that fancy code is no longer needed.

---

The number of bugs in your code is linearly proportional to how much code you have.
So less code is always better, if you can manage it.

Adding infrastructure to your code can mean faster turnaround times for changes you expect you'll need to make.
But adding infrastructure also means more code and more bugs, so adding more isn't always great.
Then when is it a good idea?

One heuritic: the code that has had the most (bugs, requirement changes, feature requests) is the most likely to have more (bugs, requirement changes, feature requests) in the future.

So it's okay to start with a 'simple' draft for everything, and add infrastructure as soon as you see a recurring pattern of bugs and changes in a particular area of your codebase.

Beyond that, you just need some experience and some good judgment ;-)

## Whether to Guess

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

