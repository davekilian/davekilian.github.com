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

---

Typical guidance for when to rewrite is over-complicated.
Remember our end goal?

> Write software that solves a problem,<br>
> does exactly what you mean it to,<br>
> and takes as little time to develop as possible

For now, let's take for granted that rewriting is a purely technical decision, that it doesn't affect your end product's interface or feature set.
Thus we can safely say rewriting doesn't affect whether our software solves the problem at hand, because rewrite or no, from the user's point of view the software ought to work the same.

That leaves us with variables to minimize:

* How long it takes to write it and fix bugs we introduced
* The number of bugs still around when we decide to ship

A tall order, no doubt.
But we can simplify the decision.
Say if you don't rewrite, the time it'll take to do new features is 

* (time needed to get the feature working on the existing codebase) +
* (time needed to fix bugs from those changes)

If you do rewrite, you're dealing with a longer lists of time costs:

* (time needed to rewrite the thing) +
* (time needed to fix bugs from those changes) +
* (time needed to add the feature after rewriting) + 
* (time needed to fix bugs from those changes)

To make the decision of whether to rewrite, just sum both lists and choose whichever is less total time.
If there are additional time costs to consider, just expand the lists to account for everything.

## Are You Gonna Need It?

Crack open a book like [Design Patterns](TODO link) and you might think software design is all about using programming constructs and abstraction to write elegant, extensible code that any developer can easily update to roll in new features and fix bugs.
Right?

Nope!

All that fancy code is actually pretty rare, and for good reasons.
Most developers eventually realize that truly clever engineering isn't fancy code, but rather recasting the problem so fancy code is no longer needed.

That said, when is fancy code really the right answer?

When you come down to it, fancy code is a betting game.
You're doing work now under the assumption that certain kinds of changes will be more likely than others in the future.
If you were right, those changes will be a cinch!

But if you were wrong, the changes will be a much bigger burden to work around than if you didn't have them in the first place.

---

Add here what was previously the design speculation section

In the end, after declaring just how sure you need to be that fancy code will be worth it, provide a heuristic:

> The code that has had the most (bugs, requirement changes, feature requests) is the most likely to have more (bugs, requirement changes, feature requests) in the future.

But by default, the answer really is fancy code is never excusable until it's proven necessary.
And even then, true cleverness is recasting the problem so that fancy code is no longer needed.

---

TODO I also want to redraft a little so that we take for granted that you're solving the problem at hand at the entire article's scope, and scope down tradeoffs for technical decisions to just fewest bugs and shortest time. We already wrote that for the first subsection, we just need to work it into the top-level scope.

---

The number of bugs in your code is linearly proportional to how much code you have.
So less code is always better, if you can manage it.

Adding infrastructure to your code can mean faster turnaround times for changes you expect you'll need to make.
But adding infrastructure also means more code and more bugs, so adding more isn't always great.
Then when is it a good idea?

One heuritic: the code that has had the most (bugs, requirement changes, feature requests) is the most likely to have more (bugs, requirement changes, feature requests) in the future.

So it's okay to start with a 'simple' draft for everything, and add infrastructure as soon as you see a recurring pattern of bugs and changes in a particular area of your codebase.

Beyond that, you just need some experience and some good judgment ;-)

## Whether to Guess

Adding infrastructure is actually speculating in your software design.

You can design up front to accommodate change, but you can't accommodate every possible change.
One way or another, you're going to need to make educated guesses about what changes are likely, to accommodate for them in advance.
(That's why the 'senior' developers on your team or usually more heavily embroiled in the business and user scenarios, whereas more junior developers often stick to the code itself).

The thing about speculating is, once it comes time to make changes to the code:

* If you were right, the change will be easy
* If you were wrong, the change will be very expensive

For example, let's say if you speculate correctly, then when the time comes to make a change, it'll only take $\frac{1}{10}$th the time it would have taken had you not acommodated for the change up front.
Let's also say if you're wrong, then when the time comes to make the change, it will take 10 times as long.

Then to break even, what's the necessary probability you speculated correctly?

$p(\frac{1}{10}x) = (1 - p)(10x)$

$\frac{1}{10}p = (1 - p)(10)$

$\frac{1}{10}p = 10 - 10p$

$10.1p = 10$

$p = .99$ (roughly)

So, you want to add infrastructure; are you 99% sure you're gonna need it?

Another facet to consider: the 10X slowdown from speculating wrong is a way bigger hit than the 10X speedup from speculating right.
Using the numbers above,

* If making the change would take 1 hour had you not speculated,
* Then the change would take 6 minutes if you had speculated (you saved 54 minutes)
* And the change would take 10 hours if you had speculated wrong (you lost 9 hours!)

In light of this, experienced developers often keep it simple by default and leave adding complexity to the future.
This means future changes aren't free, but they usually don't require expensive reworks either.

Over the long run, how often do you have to speculate correctly to break even with never speculating at all?

$1x = (\frac{1}{10}x)p + (10x)(1 - p)$

$1 = (\frac{1}{10})p + (10)(1 - p)$

$1 = (\frac{1}{10})p + 10 - 10p$

$9 = 9.9p$

$p = .90$ (roughly)

So, do you speculate correctly 90% of the time?

Keep it simple and avoid speculating if you can -- because the odds are against you!

