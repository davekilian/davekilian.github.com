---
layout: post
title: How to Evaluate Designs
author: Dave
draft: true
---

Brain dump:

Motivation: change is inevitable, even the most perfect code is subject to imperfect requirements (the real world is messy). A good design

- requires little time to make the most expected changes
- requires little time to validate changes

This would allow you to move faster and break fewer things.

Here's a quick procedure

1. Look at the requirements, and find the weakest or most likely to change. Deduce from that what is most likely to change in your code. Make a ranked list of most likely changes

2. For each change, answer the following questions:
- how many separate locations in the codebase need to be updated to make this change? Fewer is better
- what tests do you need to run to be sure ALL affected code (direct or indirect) is working as intended, no regressions? Fewer is again better

3. As time goes on, you'll see whether you were right about the changes you expected the most. Keep the list up to date as changes do actually happen, and keep re-evaluating your design to see if it is keeping up or needs some changes

A lot of good design advice falls out of these principles:

- "don't repeat yourself" means factoring so fewer code locations need to be updated for each change.
- isolation, abstraction, and focus on decoupling via code contracts are all means of controlling side effects of a code change. Limiting the scope of side effects limits the number of code paths which must be tested after a change
- testing automates validation of important code paths, allowing you to spend less time validating code that changes often, or otherwise would be costly to manually test
