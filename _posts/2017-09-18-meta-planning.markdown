---
layout: post
title: Meta-Planning
author: Dave
draft: true
---

Brain dump:

What is software? Software is a plan. An absurdly specific plan intended for computers, which do not have common sense and cannot correct the plan or improvise. But a plan nonetheless. Where do you think the word "program" comes from?

What is a plan? A sequence of steps, but not every sequence of steps is a plan. A plan must have a goal which the steps accomplish, and oftentimes plans may need to be written with variables which are filled in at the time of execution.

Why plan? (Instead of just jumping in and doing.) Historically, it's because undoing mistakes is more costly when doing the thing than when planning. Developing a plan ekes out unknown unknowns more cheaply than finding them after you've started. Think how many millions of dollars would be lost if we built a bridge and then realized at the end that it could not support its own weight.

Given all this, how do you plan a software project? How do you plan to make a plan?

This is a deep and challenging question.

Well if planning ekes out unknown unknowns, and the only way to find the unknown unknowns is to plan, then we're screwed. You can't figure those out until you start developing the software, but the whole point was to discover them before writing any software.

We cannot plan out the software in full. Once you've fully planned out of the software, the resulting plan _is_ the software.

That's not to say the situation is hopeless. Writing specs are a great way to build broad strokes of the plan before filling out the details. Those broad strokes are often where the most costly mistakes lie, so finding them with a 6 page spec is less costly than finding them 2 months into development.

There's also agile development. Agile gets confused with its helper tools and methodology, but the only real purpose of agile is to meta-plan. Instead of figuring out what the software is going to end up looking like, come up with a plan to flush out unknowns. Also keep an eye on how things are going, and constantly be reevaluating whether to change the meta-plan. In a sense, agile gives up and says hey, you can't guess your way into all the unknown unknowns, and some of those may ruin your plan. So chin up, start working, and watch very carefully for when those things turn up.

But software planning probably cannot be perfect. The only way to get a perfect software plan is to build the software itself.

This also implies a reason software estimation is so maddeningly inaccurate. To estimate the time taken to complete a bridge, factor in the time needed to complete each step. But for software, you don't have the full plan until the software is done.
