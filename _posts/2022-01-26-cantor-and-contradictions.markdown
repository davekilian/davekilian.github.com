---
layout: post
title: Cantor and Contradictions
author: Dave
draft: true
---

I've been reading math books lately. Two of my recent picks &mdash; Douglas Hofstadter's *Gödel, Escher, Bach*, and Eugenia Cheng's *Beyond Infinity* &mdash; both cover Cantor's diagonalization argument.

Diagonalization tells us something interesting about infinity, but I have some doubts about the way it's usually presented. In this post, I want to show you a different way to think about infinity and diagonalization. Along the way, we'll also touch on some of the limits of proof by contradiction.

## Background

> History - cantor, year, etc
>
> For this post, I'll just call Cantor's Diagonalization Argument, "diagonalization," for brevity.

So, what does diagonalization tell us?

If you're like me, you may have learned a little about infinity in school. It's common to think of "infinity" as a number, which we usually denote using the symbol $\infty$.

One way to ask the question which diagonalization answers is: is $\infty$ really a number? Think about it this way: all the numbers you've ever written down are "finite." If a number is finite, then it describes a fixed quantity. I could count to any finite number if I wanted to. (Although it'd take a very, very long time for really big finite numbers &mdash; possibly even longer than I will live! The fact that you can count to a number, at least in principle, is what makes a number finite.)

There are lots of finite numbers. If I wanted to talk about "finity" (that is, "finite-ness", or being finite), I'd be talking about a whole class of numbers. There is no single number called "finity."

It's not too much of a stretch to imagine something you could never count to, no matter how long you tried. This 'something' would not be finite, so why not call it "infinite?" The prefix "in" means "not," so the term "infinite" really just means "not finite." If a number can be "infinite," then it must also be possible to talk about "infinity" &mdash; being infinite, or "infinite-ness."

That brings us to the question: what is infinity? Is infinity a number, or is infinity a property that lots of numbers have? That's the question that diagonalization answers. In fact, diagonalization was the first definitive answer in western mathematics!

By the way, spoiler alert: the answer that diagonalization gives us is that "infinity" does indeed work like "finity." There are lots of numbers that are infinite, so "infinity" is a property shared by many numbers. When we write $\infty$ in a formula, we should really think of that as meaning "put some non-finite number here." There's no one number that $\infty$ always refers to.

Today, we know there are infinitely many infinite numbers; diagonalization helps us find the first two. Here's how it works:

## The Diagonalization Argument

We're going to do a proof by contradiction. In a proof by contradiction, you start by guessing something must be true, and then show that guess leads us directly into something horribly, terribly wrong. Since our guess lead to something really obviously wrong happening, that must mean our guess was wrong all along!

In more detail: to do this proof by contradiction, we're going to look at the question we're trying to answer ("is there only one infinite number?"), guess at the answer ("yes, there's only one infinite number"), and show that our guess must have been wrong. Let's get cracking!

> TODO hash out the argument

## "Something's wrong, I can feel it"

What we just walked through is the standard way of presenting Cantor's diagonalization argument. Recently, I've read Cheng do it that way in *Beyond Infinity*, as does Hofstader in *Gödel, Escher, Bach*, as does the [Wikipedia article](TODO) on diagonalization (TODO fact check the last one). Note that these two books were written almost 50 years apart from one another!

Writing out the diagonalization proof as I did just now, I have to admit that something feels off. Do you feel it too?

I mean, really, what is this result actually telling us? How can it be you can't make a set of all decimal numbers between $0$ and $1$? After all, sets are just abstract concepts. Close your eyes, and try to imagine the idea of all possible decimals that all start with 'zero point something.' Did it work? It seems like we can at least abstractly conceptualize a set of all decimals between $0$ and $1$. Why can't I imagine those things all in a set? Sets are imaginary concepts after all. Is there something wrong with our definition of a set? Doesn't something just *kinda* feel off?

And actually, come to think of it, if I can't make a complete set of decimals, then what is this thing?

<center style="font-size:200%">$\mathbb{R}$</center>

You might remember this thing from late high school math. To quote [Wikipedia](https://en.wikipedia.org/wiki/Real_analysis) as of the time I wrote this article, $\mathbb{R}$ is supposedly the "uncountable set" of all real numbers, where a [real number](https://en.wikipedia.org/wiki/Real_number) is any number that can be "represented as an infinite decimal." Sounds familiar. So clearly people who work on math for a living and are familiar with Cantor diagonalization still think the set $\mathbb{R}$ of all possible decimal numbers exists. They even invented a whole field &mdash; real analysis &mdash; to study $\mathbb{R}$. They use $\mathbb{R}$ to prove other stuff. But it also seems to not really exist? What's going on here?

This brings me to a point I wanted to make about proofs by contradiction: they don't tell you a whole lot. A proof by contradiction can show you that something is wrong, but it takes work to formally pinpoint which assumption(s) lead us astray. In fact, I suspect we were jumping to conclusions earlier when we decided that "you can't make a set of all real numbers." If we keep poking, I think I can convince you the story is a little different than that.

So let's explore diagonalization a little more and see what we can find ...

## Finite Diagonalization

We were using diagonalization to help us figure out infinity, but in the end it seems we just made ourselves more confused. What if we tried taking infinity out of the picture for a second? That might help us check whether what we were doing (that is, diagonalizing) really makes sense.

What if we tried out our diagonalization algorithm on the set of 1-digit decimals? Say we had this set:

> $0.0$
>
> $0.1$
>
> $0.2$
>
> $0.3$
>
> $0.4$
>
> $0.5$
>
> $0.6$
>
> $0.7$
>
> $0.8$
>
> $0.9$

That should be all possible 1-digit decimals. (Check my work though!)

Let's try diagonalizing this. First, pick a number from the list above; any will do. That's your first number. Now, per the diagonalization algorithm, go find the first digit of the that number, and add one to it. Got it? Okay, now, if we were doing this on infinitely many infinitely long decimals like before, we'd go to the next number and examine the next digit. But this time, because we limited ourselves to 1-digit decimals, we're done!

So, did diagonalization work? Did you manage to get by diagonalization a number that wasn't already in the set? No? Huh. Hm. Uh oh.

It's not just you. When I did this, I started with $0.0$ because it was the first number in the list. I looked at its first digit &mdash; $0$ &mdash; and added 1 to obtain my first digit &mdash; $1$. At the end, my diagonal number was $0.1$. But $0.1$ is in the set? I thought diagonalization was supposed to find a number that isn't in the set?

It seems we found something that can "go wrong" with diagonalization with finite numbers. Could it be similarly wrong when we try to diagonalize infinite numbers?

Well, for now, let's keep going anyways. Maybe it'd help to get up to two digits? After all, the real diagonalization argument runs on infinitely many infinitely long decimals, and moving from one digit to two digits at least gets us a little closer to infinity, right? Let's try again:

> $0.00$
>
> $0.01$
>
> $(...)$
>
> $0.98$
>
> $0.99$

This time I'm not going to write out every single possible decimal to save space. But, imagine that all 100 numbers from $0.00$ to $0.99$ are in the set above.

Ready to diagonalize? Try it with me! I will once again start with the first number &mdash; that's $0.00$ &mdash; and I go find its first digit, which is of course $0$. I add one to that and obtain the first digit of my diagonalized number. So far I have:

<center>$0.1?$</center>

Next I'll move to the second number, which is $0.01$. I look at the second digit, which is $1$, and I add one to that and get $2$, my second digit. Now my diagonalized number is

<center>$0.12$</center>

And now I'm out of digits, so I'm done!

But, ugh ... $0.12$ ... that was already in my set. I thought diagonalization was supposed to get me a new number that isn't in my set? Oh my goodness, this just keeps getting worse and worse doesn't it?

If you like, you can try this with three digits. Or four. Or more. Eventually, it starts to get clear that there is no point this is ever going to start working. We're doomed. But maybe we're doomed in a good way! If we can figure out why we're doomed, maybe we can learn something interesting.

And in fact, I think if you look at what we're doing with finite diagonalization, we now have a big hint to what 'really' goes wrong in Cantor's proof by contradiction ...

It was the diagonal itself.

## Diagonals Can't Cover the Decimals

You see, when we try to diagonalize over a set of finite-length decimals, we always run out of digits and have to stop. That always happens *before* we have visited every number in the set. The supposedly 'new' number we constructed turns out to only be 'new' in so far as we didn't see it during the diagonal walk. But, it's always there *later* in the list, in the part our diagonal walk never was able to reach. To see what I mean, remember our examples:

Think of the 1-digit experiment. For me, my diagonal walk started at $0.0$ and then it ended right there, because I ran out of digits. It got back $0.1$, a number further down the list after the place I had to stop.

Or the 2-digit experiment. For me, my diagonal walk covered $0.00$ and $0.01$ and then ended, because I was out of digits. I got $0.12$, a number further down the list after the place I had to stop.

See a pattern?

Think about what'd happen with 3 digits. Or 4. Or infinitely many! What happens if I continue this forever? Will my diagonal walk ever catch up with the end of the list?

Here's a table to sum up what we're seeing so far:

| Number of decimal places | How many numbers there are in the set | How many numbers we include in a diagonal walk | Percent of numbers missed by a diagonal walk |
| ------------------------ | ------------------------------------- | ---------------------------------------------- | -------------------------------------------- |
| 1                        | 10                                    | 1                                              | 90%                                          |
| 2                        | 100                                   | 2                                              | 98%                                          |
| 3                        | 1,000                                 | 3                                              | 99.7%                                        |
| 4                        | 10,000                                | 4                                              | 99.96%                                       |
| $i$                      | $10^i$                                | $i$                                            | <center>&mdash;</center>                     |
| $\infty$                 | $10^\infty$                           | $\infty$                                       | Basically all                                |

Now we can see clearly how, each time we add another digit, we make some progress by adding one more number to our diagonal walk. But alas, each additional digit also makes the set *10 times bigger*. So overall, each digit we add causes us to fall further behind &mdash; really far behind!

In fact, now we can see that, even if we walk forever, the set of numbers our diagonalization 'missed' will also grow forever. The trend never reverses; no longer how long we walk, we can never catch up. We can never even stop falling further behind!

What we've done so far isn't a formal proof, but we have a high level idea for one. It seems like we've found at least two infinite numbers here:

* Our diagonal walk, if run forever, visits infinitely many numbers
* The set of all decimals between $0$ and $1$ has infinitely many numbers

These two infinite quantities are not equal: there are more decimals in the set than there are decimals visited by a diagonal walk. So it would seem there are at least two infinite numbers that are not equal: the size of the diagonal walk, and the size of the set.

This is what Cantor was talking about with "countable" and "uncountable" infinity: "countable" refers to the length of the diagonal walk, whereas "uncountable" refers to the size of the set.

> How many numbers does an infinite diagonal walk miss, anyways? If you've done some precalculus and know what a limit is, you alreeady know enough to find out yourself!
>
> Per the table above, remember that if you do a finite diagonalization on $i$-digit decimals, your diagonal crosses $i$ of the $10^i$ numbers in the set. The proportion of "numbers missed per numbers hit" would thus be $\frac{10^i-i}{i}$. If you unconstrain $i$ and thus allow for infinitely many decimal places, the proportion would be characterized by the following limit:
>
> <center>$\lim_{(i \to \infty)} \dfrac{10^i-i}{i}$</center>
>
> If limits are new or unfamiliar, this expression asks the question, "as $i$ grows unbounded, what happens to $\frac{10^i-i}{i}$?" Limits on fractions can go one of three ways:
>
> 1. If the numerator grows faster, the fraction will grow forever
> 2. If the denominator grows faster, the fraction will approach $0$
> 3. If both are proportional, the fraction will approach that proportion
>
> For us, $10^i - i$ grows asymptotically faster than $i$ (a lot faster), so we're in case 1. So this limit approaches $\infty$ as $i$ approaches $\infty$.
>
> To summarize, it seems if you diagonalize the set of all decimals, your diagonal will certainly cross infinitely many numbers, but, for each number you *do* cross, you will have missed infinitely many more!

## Cantor, Correctly

If I'm going to claim I don't like the way Cantor's diagonalization argument is usually presented, it seems only fair to propose a better way.



> TODO Recast the proof. Basically, it's 
>
> Say you have a set of $R$ all real numbers over some interval, e.g. $[0, 1)$
>
> Represented as infinite decimal expansions
>
> Let $\infty$ be the length of an infinite decimal expansion
>
> Then $|R| > \infty$. Proof:
>
> I can diagonalize and obtain a number that I never saw once during diagonalization
>
> Diagonalization algorithm as before.
>
> Diagonalization covers $\infty$ elements of $R$.
>
> But diagonalization produces a number distinct from what it covered
>
> Therefore the diagonal didn't hit every number
>
> Therefore the set is not 'square.' It's taller than it is wide.

## More on Contradictions

> magine that you work at the toast factory; bread goes in, butter goes in, heat is involved, toast comes out. Say you have two alarms: the low bread alarm tells you it’s time to go out and get more bread; the low butter alarm tells you it’s time to get more butter. One day, management decides to streamline your factory by simplifying the alarm system. They get rid of the low-bread and low-butter alarms, and replace it with the “everything is ok” alarm. Now think: you’re on the factory floor one day and the alarm goes silent. What do you do? You know not everything is ok, but what’s wrong? Low bread? Low butter? Both? Proofs by contradiction work like the everything-is-ok alarm. They show that, of all the assumptions you made, they’re not all compatible with one another. But which is the odd one out? Or are there multiple odd ones out? You just don’t know. Proofs by contradiction are often easier to arrive out than more ‘direct’ proofs, but the flip side is, they’re usually just a start. If you want to explore what exactly it is that’s wrong, you need more investigation and a different kind of proof - like the one we sketched for [un]countable infinities here.

## Beyond Infinity

> Alephs?
>
> Plug the book I liked, also a good time to cover anything else worth saying about the 'weirdness' of infinity, or having to poke at it indirectly using finite brains and finite math.
