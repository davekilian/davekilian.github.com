---
layout: post
title: The "Not Invented Here" Anti-Pattern
author: Dave
draft: true
---

I recently read an article posted on Hacker News called
"[Center Wheel for Success: "Not invented here" syndrome is not unique to the IT world](https://queue.acm.org/detail.cfm?id=2559901)."
In this article, Poul-Henning Kamp offers a possible explanation for the
prevelance of "Not Invented Here" (or NIH) Syndrome in software engineering
projects.
If software abstraction makes it so easy to leverage other people's work, why
do developers go and reinvent the wheel?

His main argument is that we tend to use computers to automate away human
labor, instead of trying to reduce or remove that labor altogether.
Although replacing human computation is exactly what computers were designed to
do, sometimes the correct answer is to assess the fundamental process rather
than removing humans from the equation.
For HealthCare.gov as an example, he writes:

> I'm beginning to think that the reason our big IT projects sink is that we 
> make the same kind of mistake: mindlessly replacing human labor with 
> technology instead of solving the actual problem.

Although this is a reasonable argument for the examples given in the article,
I disagree that this is the heart of NIH syndrome.
Today, "NIH" is an epithet implying that someone should have used an
off-the-shelf third party component rather than reimplementing it.
Although software reuse is generally considered a good thing, there are
situations where it's not a good idea, and I believe "NIH" is a result of not
understanding the tradeoffs between using a standard component versus writing
your own.

## Inventing It Here

There are several simple reasons a developer may not use a standard
off-the-shelf component, including

* Not knowing the standard implementation exists
* Wanting to reimplement it anyway for the fun or learning experience
* Being prevented from using the component due to licensing restrictions

However, these are not interesting reasons why knowledgeable engineers choose
to reinvent the wheel for professional software products.
Let's focus on those instead.

### Matching Implementation to Requirements

When you take an off-the-shelf component, you'll often find yourself needing to
either wrap parts of the component to make it better fit your needs, or to
change your entire architecture to better incorporate the component.
The work needed to integrate the component into your architecture is roughly
unknown until you sink time learning the component's interface deeply.

When you write your own software component as part of a larger project, you're
designing it to fit snugly into your project's architecture.
Although you pay the price of implementing the component, the integration cost
should be low in general, because you're designing the interface up front to be
easy to integrate into your project specifically.

### Better Performance

This is really the previous section in a different light.
Designers for third-party components have to design the component to work well
in a variety of situations; your particular needs will likely be narrower.
With simplifying assumptions, you can sometimes satisfy your own needs with
a simpler and faster algorithm.

### Maintainability Costs

All software has bugs.
If you write a software component, you own it, and can change it to suit your
needs however and whenever you wish.
This gives you a lot of flexibility when to fix bugs when you find them.

When you depend on a third-party component that has bugs, however, the process
for resolving those bugs is not so straightfoward for several reasons:

* If the component is closed-source, you rely on the component owner to fix the
  bug.
  Closed-source projects usually have professional support, but turnaround
  isn't guaranteed to be as quick as you might like,

* If the component is open-source, you may not be able to patch the bug
  yourself due to licensing restrictions, or because the component owner may
  refuse to help troubleshoot the software once it contains nonstandard code.

* For certain classes of design problems, resolving the bug may require a
  breaking change, in which case the fix won't be rolled out until the next
  major release of the library.

On the flip side of the last point, actively maintained third-party components
tend to introduce breaking interface changes.
You're on the hook for regularly integrating upgrades of your dependencies,
as vendors frequently refuse to troubleshoot or patch very old versions of
their software.

### When It's the Core Objective

You can think of picking up a dependency on a third-party library as a form of
outsourcing.
The only difference is in outsourcing, you pay to have the component created
for you instead of having it readily available.

There's plenty of advice online warning against outsourcing primary features of
your business.
In general, if you're outsourcing core pieces of your business, you're giving
that part of your business to people who aren't aligned with you and your
product vision, which leads to friction, inflexibility and a degree of
corner-cutting.
Additionally, if you blatantly depend on another business's product, you're
exposed to the other business's whims, and you risk being commoditized into a
feature of the very product you depend on.

Typical advice is to develop your own software for your own core competency.
This applies to outsourcing and to picking up open source dependencies.

## Picking Up Dependencies

## Understanding the Tradeoffs

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
