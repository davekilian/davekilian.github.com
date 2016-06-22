---
layout: post
title: Software Design by the Numbers
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
Finishing the job here means the whole job, beyond just getting the golden path working.

Say if you don't rewrite, the time it'll take to do the feature is 

* (time needed to get the feature working on the existing codebase) +
* (time needed to stabilize)

If you do rewrite, you're dealing with a longer lists of time costs:

* (time needed to rewrite the thing) +
* (time needed to stabilize the thing after the rewrite) +
* (time needed to add the feature after rewriting) + 
* (time needed to stabilize new feature)

To make the decision of whether to rewrite, just sume both lists and choose whichever takes less time.

## When to Add Infrastructure

## Design Speculation

---

Brain dump: let's unite all the semi-mathy posts under one banner

## When to Add Infrastructure

Basically, how to design a simple software codebase without unneeded complexity.

One helpful heuristic: the code which has had the most (bugs | requirement changes) is the most likely to have future (bugs | requirement changes).
So it's okay to start with a 'simple' draft for everything, and add infrastructure or abstraction or indirection or whatever as soon as you sense a pattern of recurring bugs and changes.
Doing so helps you make quicker and more reliable bug fixes and requirement changes.

Beyond that, you just need some experience and some good judgment!

## Deisgn Speculation

The one thing that makes software design hard is change. You can design up
front to accommodate change, but you can't design up front to accommodate every
possible change. So you have to speculate what changes will be likely and
useful to accommodate for. THis is why your 'senior' developers are heavily
embroiled in the business and user scenarios, where junior developers are often
stuck looking just at the code.

If you speculate and you're right, the change will be easy.
If you speculate and you're wrong, the change will be very expensive.
If you don't speculate and keep is straightforward instead, you can avoid very
expensive reworks later on (they're not free, but they're less expensive).

Experienced developers keep it simple by default and add complexity to futre
work only when there's really good evidence to support that.

There's a fun little probability argument you could make here with
expectations: say you're going to add an abstraction speculating on future
work. If you're right, the future work takes 1/10 as long, and if you're wrong
the twork takes 10x as long.

To break even, what's the necessary probability you speculated correctly?

1x = p(1/10x) + (1 - p)(10x)

= p(1/10) + 10 - 10p

9 = 9.9p

p = .90 (roughly)

So, are you 90% certain you're gonna need it?
