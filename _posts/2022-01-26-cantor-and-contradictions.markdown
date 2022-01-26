---
layout: post
title: Cantor and Contradictions
author: Dave
draft: true
---

I've been reading math books lately. Two of my recent picks &mdash; Douglas Hofstadter's *Gödel, Escher, Bach*, and Eugenia Cheng's *Beyond Infinity* &mdash; both cover Cantor's diagonalization argument.

Diagonalization tells us something interesting about infinity, but I have a problem with the way it's usually presented. In this post, I want to show you a different way to think about infinity and diagonalization. Along the way, we'll also touch on some of the limits of proof by contradiction.

## The Diagonalization Argument

> History - cantor, year, etc
>
> For this post, I'll just call Cantor's Diagonalization Argument, "diagonalization," for brevity.

So, what does diagonalization tell us?

If you're like me, you probably learned a little about infinity in school. It's common to think of "infinity" as a number, which we usually denote using the symbol $\infty$.

One way to ask the question which diagonalization answers is: is $\infty$ really a number? Think about it this way: all of the numbers you've ever written are "finite." If a number is finite, then it describes a fixed quantity; if I wanted to, I could count to any finite number, even if it'd take a very, very long time for really big finite numbers. So in that way, "finity" (that is, finite-ness) isn't a number; it's a property shared by many numbers.

It's not too much of a stretch to imagine something you could never count to, no matter how long you tried. This 'something' would not be finite, so we'd call it "infinite," since the prefix "in" means "not." So then, what is infinity? Is infinity a number, or is infinity ("not-finite-ness") a property that lots of numbers have? That's the question that diagonalization answers. 

Spoiler alert: the answer is the latter. "Infinity" is a property shared by many numbers. In fact, we now know there are infinitely many infinite numbers; diagonalization just helps us find the first two. Here's how it works:

We're going to do a proof by contradiction. That means we're going to look at the question ("is there only one infinite number?"), guess the answer ("yes, there's only one infinite number"), and show that our guess must have been wrong.

> TODO hash out the argument

## "Something's wrong, I can feel it"

What we just walked through is arguably the standard way of presenting Cantor's diagonalization argument. Recently, I've seen Cheng do it that way in *Beyond Infinity*, as does Hofstader in *Gödel, Escher, Bach*, as does the [Wikipedia article](TODO) on diagonalization (TODO fact check the last one).

But I have to admit that something feels off. What do you mean you can't make a set of all decimal numbers between $0$ and $1$? After all, sets are just abstract concepts. Close your eyes, and try to imagine an infinitely huge array of decimals that all start with 'zero point something.' Did it work? It seems like we can at least imagine the decimals between $0$ and $1$. Why can't a set of those exist? Is there something wrong with our definition of a set? Something feels off, doesn't it?

And actually, come to think of it, if I can't make a complete set of decimals, then exactly is this thing?

<center style="font-size:200%">$\mathbb{R}$</center>

To quote [Wikipedia](https://en.wikipedia.org/wiki/Real_analysis) as of the time I wrote this article, $\mathbb{R}$ is supposedly the "uncountable set" of all real numbers, where a [real number](https://en.wikipedia.org/wiki/Real_number) is any number that can be "represented as an infinite decimal." Sounds familiar. So clearly people who work on math for a living and are familiar with Cantor diagonalization still think the set $\mathbb{R}$ of all possible decimal numbers exists. They've invented a whole field &mdash; real analysis &mdash; to study $\mathbb{R}$. They even use $\mathbb{R}$ to prove other stuff. What's going on here?

This brings me to the point I wanted to make about proofs by contradiction: they don't tell you a whole lot. A proof by contradiction can show you that something is wrong, but it takes work to formally pinpoint which assumption(s) lead us astray. In fact, I suspect we were jumping to conclusions earlier when we decided that "you can't make a set of all real numbers." If you keep poking, the real story is a little different.

So let's explore diagonalization a little more and see what we can find ...

## Finite Diagonalization

We were using diagonalization to help us figure out infinity, but it seems we just made ourselves more confused. What if we tried taking infinity out of the picture for a second? That might help us check whether what we were doing (that is, diagonalizing) really makes sense.

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

Let's try diagonalizing this. First, pick a number from the list above; any will do. That's your first number. Now, per the diagonalization algorithm, add one to the first digit of the first number. Got it? Okay, now, if we were doing this on infinitely many infinitely long decimals like before, we'd go to the next number and examine the next digit. But this time, because we limited ourselves to 1-digit decimals, we're done!

So, did diagonalization work? Did you manage to get by diagonalization a number that wasn't already in the set? No? Huh. Uh oh.

It's not just you. When I did this, I started with $0.0$ because it was the first number in the list. I looked at its first digit &mdash; $0$ &mdash; and added 1 to obtain my first digit &mdash; $1$. At the end, my diagonal number was $0.1$. But $0.1$ is in the set? I thought diagonalization was supposed to find a number that isn't in the set?

This is our first clue that, whatever goes wrong with diagonalization, might not just have to do with infinity. We tried it on a finite set and it still didn't work the way we expected.

But, let's keep going anyways. Maybe it'd help to get up to two digits? After all, the real diagonalization argument runs on infinitely many infinitely long decimals, and moving from one digit to two digits at least gets us a little closer to infinity, right? Let's try again:

> $0.00$
>
> $0.01$
>
> $(...)$
>
> $0.98$
>
> $0.99$

This time I didn't write out every single possible decimal to save space. But, imagine that all 100 numbers from $0.00$ to $0.99$ are in the set above.

Ready to diagonalize? Try it with me! I once again start with the first number &mdash; that's $0.00$ &mdash; and I go find its first digit, which is of course $0$. I add one to that and obtain the first digit of my diagonalized number. So far I have:

<center>$0.1?$</center>

Next I move to the second number, which is $0.01$. I look at the second digit, which is $1$, and I add one to that and get $2$, my second digit. Now my diagonalized number is

<center>$0.12$</center>

But, ugh ... $0.12$ ... that was already in my set. Oh my goodness, this just keeps getting worse and worse doesn't it?

If you like, you can try this with three digits. Or four. Or more. Eventually, it starts to get clear that this isn't ever going to start working. But more importantly, we've gotten a big hint as to what was 'really' wrong all along ...

It was the diagonal itself.

## Diagonals Can't Cover the Decimals

You see, when we try to diagonalize over a set of finite-length decimals, we always miss some elements of the set. Every time, we terminate our diagonal walk because we ran out of digits, even though there were more numbers further down the list. And, we keep finding that the 'new number' that our walk gave us was *always* somewhere in the list, but it was *never* one of the numbers we visited during the walk. That is, diagonalization produces a number it has never seen, but we always have that number somewhere else in the list.

That's the key problem with diagonalization: it's not that it provides a number that wasn't already in the set; rather, it's just not possible to walk diagonally over every number in the set. Diagonal walks always terminate because they run out of digits, before visiting every number in the set. Then maybe we shouldn't be so surprised that the 'new number' we end up with is always just one of the numbers we never got a chance to visit.

Think of the 1-digit experiment. For me, my diagonal walk started at $0.0$ and then it ended because I ran out of digits; it produced $0.1$, a number further down the list.

Or the 2-digit experiment. For me, my diagonal walk covered $0.00$ and $0.01$ and then ended, because I was out of digits. It produced $0.12$, a number further down the list.

Going back to Cantor's original argument, what happens if I continue this process forever? Maybe a finite walk and an infinite walk aren't the same thing after all. Surely, if we run the diagonal walk forever, we eventually have to cover every number in the set, right?

Well, it turns out no, the diagonal walk does not have any hope of catching up.



> TODO I was originally going to have set construction running with diagonalization in tandem, but that requires some proof that running the two together really 'makes sense.' That seems like it's hard to explain and plus I don't really need to anyways. All I need to show is that, for a set of $i$-digit decimals, a diagonal walk visits $i$ numbers and there are $10^i$ numbers in the set. The longer you continue the walk, the further behind you fall.
>
> TODO conclude with the 2D array of digits metaphor again. But now, it's actually 'countably' infinite digits horizontally, vs 'uncountably' infinite numbers vertically. It's an infinite rectangle, infinitely tall and infinitely wide; but at the same time, it's also infinitely taller than it is wide. That's why the diagonal as we thought of it is guaranteed not to cover all the numbers.
>
> TODO the conclusion: we can of course imagine a set $\mathbb{R}$ of uncountably infinitely many decimal numbers. You can try to diagonalize this set, but you'll always miss some; the number you get from diagonalizing is just one of the many numbers you missed. 

## How many numbers did we miss, anyways?

> TODO in fact, if you've done precalc, you can characterize what's going on with a limit: at step i there are 10^i numbers overall, and we've covered i of them. So the ratio is 10^i / i. If you take the limit as i approaches infinity, 10^i dominates i asymptotically, and you get infty.
>
> Which is kind of startling: it turns out, even if we run an infinite diagonal walk, even though we visit $\infty$ numbers during our walk, we miss $\infty$ numbers per every number we do hit!
>
> Alephs.

## More on Contradictions

> magine that you work at the toast factory; bread goes in, butter goes in, heat is involved, toast comes out. Say you have two alarms: the low bread alarm tells you it’s time to go out and get more bread; the low butter alarm tells you it’s time to get more butter. One day, management decides to streamline your factory by simplifying the alarm system. They get rid of the low-bread and low-butter alarms, and replace it with the “everything is ok” alarm. Now think: you’re on the factory floor one day and the alarm goes silent. What do you do? You know not everything is ok, but what’s wrong? Low bread? Low butter? Both? Proofs by contradiction work like the everything-is-ok alarm. They show that, of all the assumptions you made, they’re not all compatible with one another. But which is the odd one out? Or are there multiple odd ones out? You just don’t know. Proofs by contradiction are often easier to arrive out than more ‘direct’ proofs, but the flip side is, they’re usually just a start. If you want to explore what exactly it is that’s wrong, you need more investigation and a different kind of proof - like the one we sketched for [un]countable infinities here.

## Beyond Infinity

> Plug the book I liked, also a good time to cover anything else worth saying about the 'weirdness' of infinity, or having to poke at it indirectly using finite brains and finite math.
