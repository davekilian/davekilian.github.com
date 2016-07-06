---
layout: post
title: Fundamentals of Software Design
author: Dave
draft: true
---

There's lots of great advice out there about how to design software, and so much of it is just a few keystrokes away.
The only problem with all this information is the ensuing overload.
Spend a few hours reading all that advice, and you might get the impression that designing software is complicated!

When information overload is a problem, core guiding principles are often the solution.
I believe that the core guiding principle of software design is that, with every choice, we try to minimize two variables:

* **How many problems the choice will introduce**, from the simplest of bugs to the grandest-scale misunderstanding of user needs.
* **How much time it'll cost**, not just right away to ship the feature, but also the ongoing cost it takes to maintain your code and adapt it to future demands.

These two variables are sometimes at odds, which gives us some flexibility in deciding how to trade off between the two.
Depending on the situation, a small decrease in quality that yields bountiful time savings sometimes makes sense, but not always (e.g., if you plan to launch your code into outer space).

So in short, we're dealing with a simple cost-benefit tradeoff.
The cost is developer time, and the benefit is a better product.
We want the highest quality software in the least time, and all our decisions serve that goal.

Does that seem too obvious to be useful?
Then for the rest of this post, let's show how these fundamental principles give rise to all sorts of common engineering advice.

## What Features Belong in Version 1?

When you're starting out a new project, one question that soon arises is which ideas to code up first.
The commonly accepted answer to that question is: start with absolutely no features now that you could do later.
The result should be the minimal design that accomplishes the goal: a.k.a. the Minimum Viable Product.

Why?
A commonly cited reason is agility: a smaller set of features means a simpler product that you can iterate on more quickly.
Quick iterations mean you can revise the core idea behind your product until you're sure you have a winner.
In other words, for each feature we decide not to do up front,

* We do not make the product unusuable (else we would have done it)
* We save time we would have needed to develop the feature
* We save time we would have needed to maintain the feature as our designs iterated

We have arrived back at our original principles: the quality-time tradeoff.
In this case, by deciding to pare down the feature list, we take a small quality hit (missing features) for huge time savings (no maintenance on features we didn't do).

> ### Agile Development
> 
> You can generalize this idea beyond minimum viable products.
> In general, deciding to pare down your feature set lends you **agility**, or room to iterate.
> This agility doesn't mean exactly the same thing as Agile.
> 
> So much has been written on Agile software development that the term has taken on multiple meanings of its own, typically involving tools like kanban, sprints, user stories, and the like.
> This is a tragic missed opportunity: there's something real here, but as a field we've once again missed the forest for the trees.
> You can even see this going all the way back to the [Agile Manifesto](http://www.agilemanifesto.org/) that started it all.
> 
> Forget the zeitgeist around Agile for a moment.
> Have you ever stopped to consider why the word "agile" was used to describe that style of software development, and not another word?
> The answer lies in the point we just made in the previous section:
> 
> > All the code you don't write now is time you can spend changing your mind later.
>
> That's the idea: agility is the ability to change your mind quickly.
> If you go back and reread the manifesto, you'll notice 'principles' 3-12 are all means to one end: time efficiency.
> We could summarize those twelve princples in three bullets:
> 
> 1. We want to maximize the usefulness of the product
> 2. We believe we can only do so by experimenting and redrafting
> 3. We focus on time efficiency, so we can redraft more
>
> Beyond that, Agile tools and methodologies are intended to help maximize time efficiency.

## Should We Outsource?

One thing programmers learn early on is not to reinvent the wheel.

Every programming language has some notion of a 'library' or 'module' that allows you to reuse code in multiple projects.
Plenty of code is available online for dropping into your project; some is free, some is provided for pay.
The question is, when should you pick an off-the-shelf solution instead of rolling your own?

Keep in mind that our goal is to develop the best product with the least development cost.
Also keep in mind that our requirements are the same whether or not we pick up an off-the-shelf solution.
This allows us to simplify the question:

> Will outsourcing decrease the overall time cost for developing and maintaining my product?

When answering this question, there are two sorts of time costs to consider: costs up front, and long-term maintenance costs.
Up-front costs include researching the options, selecting one, learning how to use it, and actually integrating it into your project.
Further costs down the line include fixing bugs you find in the library, and/or working around issues the maintainer will not fix at the library level.

Common advice for ingesting off-the-shelf components lines up with minimizing these costs:

* **When selecting a component to use, default to the industry standard.**
  The industry standard component will likely be well-documented, have an active and responsive crew of maintainers, and will have few bugs that other people haven't already found.

* **If you're not selecting a popular, industry-standard component, have a maintenance plan.**
  What are you going to do if you find a bug, and the maintainer isn't willing to help you fix it?
  Are you going to work around issues at your layer?
  Will you have the source code and be able to fix it yourself?

* **Good things to outsource are complex and error-prone**. 
  Things that involve complex specifications and tricky parsing logic (like HTTP, or HTML) are likely to take a long time to fully develop and maintain.
  Given the 'roll your own' option has high costs, the 'off the shelf' option is likely to have lower time cost, even if you end up having to deal with bugs in external code.

* **Don't outsource your core functionality.**
  Your core features are the most likely to require iteration, and bugs in your core features are the most serious.
  By outsourcing these features, you greatly increase the time cost of iterating on your ideas, and of getting mission-critical issues fixed.
  These increased costs often end up outweighing the costs of developing those features yourself.

## Is It Time to Rewrite?

When fixing bugs, adding features, or otherwise maintaining an existing software product, a natural question that often arises is whether to rewrite a particular body of code.
This can include anything from shuffling a few lines of code around a few functions to redesigning an entire system.

Typical guidance for when to rewrite is over-complicated.
We can simplify a lot like we did with the outsourcing question:

> Will rewriting decrease the overall time cost for developing and maintaining my product?

To answer, we just need to do a couple of sums:

If we do not rewrite, then the cost of adding each feature will be:

   > How long it takes to add the new feature +<br>
   > How long it takes to stabilize the new feature

If we do rewrite, then the total cost is:

   > How long it takes to rewrite +<br>
   > How long it takes to stabilize after rewriting +<br>
   > How long it takes to add the new feature post-refactoring +<br>
   > How long it takes to stabilize the new feature post-refactoring

Once we sum those two costs, we can just pick whichever option is less cost overall.
You can factor whatever you want into the two cost lists, as long as you're fair with all your estimates.

To summarize, the total cost of rewriting and doing the feature can clock in under the cost of just adding the feature without a rewrite, but usually this only works if the rewrite is both scoped and significantly simplifies extending the code later on.
This is often the case for smaller code changes, but more care is needed when considering rewrites for large systems.

## Are You Gonna Need It?

Crack open a book like [Design Patterns](TODO link) and you might think software design is all about using programming constructs and abstraction to write elegant, extensible code that any developer can easily update to roll in new features and fix bugs.
Right?

Nope!

All that fancy code is actually pretty rare, and most experienced developers end up realizing that truly clever engineering isn't fancy code, but rather rotating the problem to the point fancy code is no longer needed.

If fancy code is rarely needed, when is it the way to go?
To answer, we'll recast the question in a way that should look familliar:

> Will adding infrastructure to accommodate change up front decrease the overall time cost for developing and maintaining my product?

This is where things get fuzzy.
Figuring out which changes to accommodate for is a betting game: you're doing work now in the hopes that certain kinds of changes will be more likely than others in the future.
If you were right, those changes will be a cinch; but if you were wrong, working around your unnecessary infrastructure will be a much bigger burden than if you hadn't done any extra work up front.
Usually the way to win the game is not to play: don't add infrastructure up front unless you are *extremely* confident the odds are in your favor.

How confident would that be? 
Let's say you're speculating on an order-of-magnitude speedup/slowdown.
In that case, if you guess correctly, then when the time comes to make a change, it'll only take $\frac{1}{10}$th the time it would have taken had you not anticipated the change up front; but if you're wrong, the future change will take 10 times as long.

Then the break-even probability that you anticipated correctly would be:

$p(\frac{1}{10}) = (1 - p)(10)$

$\frac{1}{10}p = 10 - 10p$

$10.1p = 10$

$p = .99$ (roughly)

So, you want to add infrastructure to make future changes easier; are you 99% sure you're gonna need it?

Things only get worse from there.
Say you get into the habit of speculating on future changes to your codebase.
Over the long run, how often do you have to speculate correctly to break even with never speculating at all?

$1 = (\frac{1}{10})p + (10)(1 - p)$

$1 = (\frac{1}{10})p + 10 - 10p$

$9 = 9.9p$

$p = .90$ (roughly)

So, do you speculate correctly 90% of the time?

Why are these numbers so extreme?
The problem is that we're dealing with order-of-magnitude speedups and slowdowns, and an order-of-magnitude slowdown is way more significant than an order-of-magnitude speedup.
Using the numbers above,

* If making the change would take 1 hour had you not speculated,
* Then the change would take $\frac{1}{10}^{th}$ that, or 6 minutes, if you had speculated (you saved 54 minutes)
* And the change would take $10X$ that, or 10 hours, if you had speculated wrong (you lost 9 hours!)

In light of this, experienced developers often keep it simple by default and leave adding complexity to the future.
This means future changes aren't free, but they usually don't require expensive reworks either.
This is also the reason why 'senior' engineers tend to be more embroiled in the business itself whereas 'junior' engineers tend to focus mostly on code: understanding the business gives you better data to speculate on.

If all this seems overwhelming, there's an easy way out: just don't speculate.
Keep the code simple and straightforward, and don't add infrastructure until you already know you need it.
A useful heuristic is to remember the code that's most likely to need work in the future is the code that has needed the most work in the past.
So start simple, and expand out as the need becomes clear.

Keep it simple -- because the odds are against you!

> ### Aside: Wanna Be a "10X" Developer?
>
> Open secret: when it comes to speculating on future code changes, pretty much everybody is wrong pretty much 100% of the time.
> You can't foresee bugs: if you could, you wouldn't make bugs in the first place.
> And you're can't foresee feature requests: there are just too many possibilities to choose from.
>
> Another open secret: most people speculate on future code changes all the time.
> Why wouldn't they?
> What's the point of that fancy computer science education, and reading all that literature on design, if in the end you should always stick to what you learned in CS 101?
>
> Consider this: if you believe the back-of-the-napkin numbers we made up above, then most people speculate incorrectly on future code changes nearly 100% of the time, making them $\frac{1}{10}^{th}$ as effective as they could be if they didn't speculate.
> Wanna be 10X as effective as these people?
> Just don't speculate!

