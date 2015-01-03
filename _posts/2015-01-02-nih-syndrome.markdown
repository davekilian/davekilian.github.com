---
layout: post
title: Not Invented Here Syndrome
author: Dave
draft: true
---

## Brain Dump

Why is Not Invented Here syndrome so prevalent in software engineering? 
 
* Inexperience probably plays a part, but lets talk about when this is not the case 
 
* It's often more fun to roll your own as a dev, bug lets talk rationalizations instead 
 
* Lets ignore the 'free' element of open source, since the tradeoff of getting a component for free is needing to pay someone (in or out of house) to support it  
 
* Rolling your own lets you iterate and redesign with ease, not a bad idea for your secret sauce / custom code / the main implementation that does the thing your product does. Because that's the code that will need to iterate along with your product vision 
 
* I think a big part is maintenance and supportability: 

    1. Code you wrote yourself has shortcomings and defects you are aware of. Bug reports can go to a member of your team rather than to an open source project over which you don't have much influence 

    2. Many open source projects have weak or nonexistent support guarantees. You will end up needing to train a resident expert anyways 

    3. Nobody will go and change the API of that component without you being aware of it. Breaking changes in support libraries can be costly for software projects, but refusing to upgrade can leave your project in an unsupported scenario 

    4. _In certain cases_, rolling your own makes it harder to exploit your software. Think about vulnerabilities in popular projects, or tampering / back doors in smaller projects. Of course, for certain software components, using a well reviewed off the shelf component is still more secure. 
 
Given all these pitfalls, why do people cry foul (or "NIH") to people who decide to roll their own? 

* Certain components are incredibly complex to get right, and benefit from peer review. Unless you can hire a retainer of domain experts to review these components, you're better off using some standard and highly vetted. Examples include container/collection classes and code that handles secure data/channels. 

* Other components require a ton of code and man hours to stand up, but don't benefit your project any more than a standard off the shelf component. Attempting these in your own project can lead to corner cutting or the project being cut due to overruns 

* Using off the shelf components often allows you to hire developers who already know the technology, with little to no training required. It also makes it easier for developers to find help online when they hit roadblocks. For developers, it also makes the skills they learn in the job more broadly applicable. 

Given these pros and cons, when should you outsource? 
 
Basically always, unless on of the following applies 

* It's simple to write your own, and you're likely to get it right the first time 

* There is no off the shelf component for what you need (in which case, consider publishing one!) 

* This is at the core of what your project does 
 
For the last point, remember to keep things in perspective. Your team might own and develop an in house build system for a major website, but in the end the website is the product in question, not the code your team owns. In this example, it is perfectly reasonable to appropriate a standard build system and flesh it our for your projects specific needs. 
 
Final corollary: Treat outsourcing the same way as using of the shelf components. Think of outsourcing as paying out of pocket to create a new off the shelf component
