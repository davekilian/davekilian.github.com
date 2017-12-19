---
layout: post
title: Mainstream Niche Programming Languages
author: Dave
draft: true
---

The title of this post is not a typo!
Some programming languages are "mainstream," in that they are used widely, but also "niche," in that there are few if any competing programming languages which fit the same use case.

This situation is not all that common.
In most cases, these languages have (or once had) some artificial limitation which prevented other new languages from taking hold in the same space.
As time goes on, it gets harder and harder to displace these languages, as more code gets written in these languages and more optimizations are implemented in their compilers.

This has happened at least twice as far as I know:

One example is is **JavaScript**.
Originally, JavaScript was hard to displace because it was (a) the only language implemented by browser vendors, and (b) simple enough to make it feasible to implement and ship with a browser.

Another example is **C++**, which fills the niche "object-oriented, direct memory access, no garbage collection" (a set of features commonly desired in large-scale systems software like databases and network servers).
It's significantly difficult to displace C++ because figuring out how to make objects usable without garbage collection is an unsolved problem in the field.



---

There are some programming languages which fill a specific niche, but are nonetheless popular and used widely.
These languages are in the odd position of being used widely, yet specific enough to have little, if any competition.

Examples include:

* **Javascript**, which fills the niche of "being able to run in a web browser"
* **C++**, which fills the niche of 

These languages are fascinating, in that they evolve 

---

The title might sound like a bit of an oxymoron, but basically, there are programming languages that fill an extremely popular 'niche' 

These languges end up looking funny. Why?
When there are no suitable alternatives, you have to solve every problem in that language, whether or not the language is good at that problem.
Ergo, you abuse the language’s syntax to achieve what you need (js classes, C++ template metaprogramming).
Inevitably, people run into trouble when abusing syntax, and go to the language committee to have their specific problems point fixed.
Some of those end up getting added to the language.
Multiply this by a lot, and you end up with weirdly complex languages with ambiguously parsed syntax/blurred lines between the language syntax and the standard library.

Despite being on somewhat opposite ends of the spectrum, C++ and JavaScript are both good examples:

* It's obvious for JavaScript: it's the only imperative language you can run in a browser
* It's true but less obvious for C++: basically, if you care about memory representation and/or need manual memory management

How can we foster greater competition in these areas?
