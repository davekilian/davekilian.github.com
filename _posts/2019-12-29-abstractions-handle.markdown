---
layout: post
title: All Abstractions Hide, Good Abstractions Handle
author: Dave
draft: true
---

Basic idea: people talk about abstractions hiding complexity, and by definition of an abstraction that's something an abstraction can do. But a better way to think about abstractions is as *handling* complexity rather than just hiding it.

Hiding a problem without handling it means you're not really hiding it at all - you end up with a leaky abstration, an inefficient/badly performing abstraction, or both.

Maybe segue into tips for designing abstractions that do a good job of handling problems in a flexible/resuable way, such as a policy-mechanism split (the abstraction provides a mechanism, and accepts policy from the caller). Bonus: unit tests are now relatively easy to write



