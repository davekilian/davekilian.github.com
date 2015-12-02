---
layout: post
title: When to Add Infrastructure
author: Dave
draft: true
---

Basically, how to design a simple software codebase without unneeded complexity.

One helpful heuristic: the code which has had the most (bugs | requirement changes) is the most likely to have future (bugs | requirement changes).
So it's okay to start with a 'simple' draft for everything, and add infrastructure or abstraction or indirection or whatever as soon as you sense a pattern of recurring bugs and changes.
Doing so helps you make quicker and more reliable bug fixes and requirement changes.

Beyond that, you just need some experience and some good judgment!

