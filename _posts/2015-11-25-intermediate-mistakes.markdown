---
layout: post
title: Common Mistakes Made by Intermediate Programmers
author: Dave
draft: true
---

Some mistakes I've come across after reading a good bit of intermediate-level
code, and some advice about how to avoid making the mistakes.

Each section should have a description of the mistake, an example, and a
discussion of how to fix the example

* The big one: speculative abstraction
    * Creating layers of indirections because you 'think' you might need them
    * You can't prove you need them, and you probably won't
    * aka: You Aren't Gonna Need It
    * A common reason for adding abstraction is to make software 'extensible'
    * But the easiest software to extend is plain code with no abstraction
    * Specific examples
        * Why should every literal be hidden in a constant?
          If a literal is used once, just let it be a literal
          If a literal is used all over, fix your code so it's only needed once
        * The 'everything is an interface' anti-pattern
        * Abusing depedency injection/inversion of control
* The minimum code spanning tree
    * Over-adhering to 'don't repeat yourself'
    * You have less code, but the code is extremely complex
    * The code is difficult to reason about
* Non-reusable abstraction
    * "This chunk of code looks complicated. Let's pull it out and make a
      method"
* Clever syntax
    * Most language syntax exists for a single killer app
    * Trying to use it for something else feels clever, but usually just harms
      readability in the end.

Takeaway: these all share an underlying trait: they're way too complex

* Software should be stable first, and readable second. All else is ancillary
* Simple software is readable
* Simple software is easy to reason about
    * Easier to catch bugs early
    * Easier to test
    * Easier to debug
* Simple software is easier to reuse
* Simlpe software is easier to refactor
* Simlpe software is easier to extend

Bonus: Take a popular language and pull out its big-name features. Then show
the killer app for which the feature is useful, and show one or more examples
of abuses.

