---
layout: post
title: Testing is Part of Your Design
author: Dave
draft: true
---

Brain dump:

Premise: the goal of a "good design" is only to limit the introduction of regressions. Sure we say a design is "easy to extend" but what we really mean is "easy to extend without messing up everything else." There are always ways to extend poor designs, it just involves a much longer stabilization period

If you believe that, then testing is part of your design.

Sometimes the best way to prevent regression is to isolate code. Sometimes it's to remove duplication. Etc

But sometimes the best way to prevent regressions is aggressive unit testing.

Examples
