---
layout: post
title: Design Speculation
author: Dave
draft: true
---

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
