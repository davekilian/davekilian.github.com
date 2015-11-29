---
layout: post
title: Floating Point is Deterministic
author: Dave
draft: true
---

Basically, floating point is deterministic.
It's just that mathematical axioms don't (quite) hold.

Start with a discussion about representation.
Use that example about the second largest possible float, and the difference between the largest and the second largest.

Then we can explore the fact that float is deterministic, but every operation introduces error.
At bigger values, the error is bigger, for aforementioned reasons.

The error causes some axioms not to hold.
Then we can walk through a few axioms and show how error terms propagate through, invalidating the axioms.


