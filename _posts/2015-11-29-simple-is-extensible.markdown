---
layout: post
title: The Simplest Code is the Most Extensible
author: Dave
draft: true
---

Sort of a followup to Fowler/MonolithFirst.

Many programmers believe you need to add core abstractions to your program to make it 'extensible.'
Must a simple program which just does its job with no extensibility is the easiest to extend.

To extend a program with existing abstractions, you often need to rework the existing abstractions to add a new one.
Sometimes this abstractions impose unnecessary constraints on your design.
This is unfortunate when the abstractions weren't necessary to begin with.

When you extend a program which is written simply with no indirection, you'll often be adding an indirection on top of existing code.
This is much easier and cleaner.

---

Lifted from a different draft post

* Software should be stable first, and readable second. All else is ancillary
* Simple software is readable
* Simple software is easy to reason about
    * Easier to catch bugs early
    * Easier to test
    * Easier to debug
* Simple software is easier to reuse
* Simple software is easier to refactor
* Simple software is easier to extend
