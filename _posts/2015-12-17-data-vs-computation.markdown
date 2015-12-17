---
layout: post
title: Data vs Computation in Programming
author: Dave
draft: true
---

Basically, earlier programming languages including most of the C derivatives
were built with computation in mind. That sounds silly, but I mean it in a
specific sense. A lot of new technology from that era was about computing
interesting results (compressing, multimedia, etc).

Today processor growth speeds have stalled, but data capacity is still on the
uptick, however slowly. Most new apps nowadays are 90% data processing, with
only a little computation on that data in between.

Data programming appears all over the place. Most web apps? Most web based
startups? Apps designed around organizing data in a clever and/or attractive
way. The main job is collect and move data round.

The recent renewed interest in function programming comes from its focus on
data, just as imperative models focus on computation. Consider: modern FP
languages typically have strong data typing and elegant ways to ferry around
data with bits of computation done on it in between. Meanwhile imperative
languages make it easy to process a data stream, but don't help you much with
moving that stream around. In fact, modern imperative languages have recently
been based on weak typing and dynamic models, having only recently started to
rebound back into the land of strong typing.

