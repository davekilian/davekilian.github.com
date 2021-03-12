---
layout: post
title: Queuing in Distributed Systems
author: Dave
draft: true
---

A basic rundown of things to know about queuing when it comes to request servers and distributed systems. I don't have a full list of topics yet, but I'm thinking of basic things like

* Basic literacy: queue depth, latency, throughput
* At QD1, latency implies throughput and vice versa. Above QD1, anything goes.
* QD1 is the fastest your system can go; above that, lock contention can start to play in
* As you hit your system's throughput limit, you get arbitrarily high queuing delays. A simple JavaScript simulator might show how this works?

I'm sure there's other stuff worth covering here as well.