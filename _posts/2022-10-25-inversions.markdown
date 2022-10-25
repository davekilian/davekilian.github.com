---
layout: post
title: "Inversions are Hard: The Monty Hall Problem"
author: Dave
draft: true
---

Inverting stuff in your head is hard: it can be done, but it never comes naturally, and you have to think *really* hard to be sure you're not making mistakes. It's the reason sentences like this feel awkward:

<center>"Why didn't he not go to class today?"</center>

... while this is perfectly clear:

<center>"Why didn't he stay home from class today?"</center>

(Did you get from the first sentence that the subject *did* go to class today? Look again!)

It's also the reason well-trained programmers, who are used to parsing complex boolean conditions and *should* know how to do this by now, still write binary search code with silly little errors and off-by-one bugs. 

And, it's also the reason people have so much trouble with the Monty Hall puzzle: a brainteaser which hinges on a well-hidden inversion. Let's talk about that.

## The Monty Hall Puzzle

Given you're reading this, the odds are good you've heard about the Monty Hall brainteaser before. If not, [the Wikipedia article](https://en.wikipedia.org/wiki/Monty_Hall_problem) gives a good overview of the puzzle, its solution, and some history behind it. Briefly, the puzzle involves a prize hidden behind one of three doors:

[ TODO a doodle ]

Say you pick one of the three doors, and then I open one of the doors that *doesn't* have a prize hidden behind it:

[ TODO a doodle ]

Now I give you a choice: would you like to stick with the door you picked, or switch to the other one? Does it make a difference? Famously, the answer is *yes*: switching doors will double your chances of winning!

Certainly unintuitive! In fact, when Marilyn vos Savant first answered this brainteaser in her weekly *Ask Marilyn* column in *Parade* magazine, she famously received [thousands of letters correcting her, and berating her for getting it so wrong](https://en.wikipedia.org/wiki/Monty_Hall_problem), even though she was the one who got it *right* &mdash; the corrections were all wrong! Among the thousands of people mailing angry (and incorrect) corrections were well educated people, PhDs, mathematicians &mdash; this problem tricked them all!

Since then, it's become something of an Internet passtime to publish high-effort, lengthy explanations of why switching doors in the Monty Hall puzzle really does double your odds of winning. Today, I'll indulge in this time-honored tradition myself; but what I really want to get to isn't why switching works, but rather why this problem is so confusing in the first place. I suspect it's because we find inversions confusing to think about: there's a key inversion hidden right in the center of the puzzle.

To see what I mean, let's look at a series of game designs. A mathematician might call all these games *isomorphic* to one another: the details are different, but at the core, they're all really the same game.

Let's start with the first variant, which is by far the simplest:

## Game 1: Choose Your Odds

Here's a simple game. Just like before, I show you three doors:

[ TODO the three doors doodle again ]

First I ask you a question: would you like me to hide one prize behind one door, or two prizes behind two different doors? You answer, and then I hide the prize(s). Then I let you pick a door and keep whatever's behind it. You pick a prize door, you win!

So, do you decide on one prize or two prizes? Two, right? You only get to pick one door, but playing a game with two prize doors means you have a 2 in 3 chance of winning; if there's only one prize, your odds are only 1 in 3 of finding it, half as good.

If this seems a little obvious then great! That just means you're smart enough to figure out this Monty Hall thing after all :-) In fact, the game I just described *is* the Monty Hall game &mdash; all I did was remove all the confusing nonsense. To show you that's true, let's start adding back the confusing nonsense bit by bit. Each step of the way, I'll show you you're still playing the game we just described above.

## Game 1(b): One Little Tweak

To start, let's play the exact same game as before, but with the steps in a slightly different order.

Once again, I show you three doors:

[ TODO the three doors doodle again ]

Now, this time I ask you to pick a door *first* &mdash; but don't open it yet! Before you open the door, you have to answer the same question as before: would you like me to hide one prize behind one door, or two prizes behind two doors? Then, based on your answer, I hide the prize(s) and let you open the door you already picked. (I promise to assign prizes to doors randomly &mdash; I promise not to avoid the door you already picked.)

You might need to squint a little, but this is indeed the same game as before: the same things happen, you get the same choices, and your decisions affect your chances of winning the same way. All that changed was the order things happen: before, you had to choose how many prizes and then pick a door; now, you pick a door and then pick how many prizes. Other than that one change, everything is the same.

Given it's the same game, you should feel safe to make the same choices! If you choose to play with two prize doors, then there's a 2 in 3 chance the door you *already* picked is a prize door &mdash; if you choose to play with just one prize door, there's only a 1 in 3 chance the door you picked will end up being the prize door. You want to play with two prize doors, right? Great! See, same game, different details.

Let's mix it up a bit more.

## Game 2: Invert or Inert?

Okay! Now let's say I'm a tricksy game show host, and I want to play this game with you, but I also want to make it as confusing as possible for you. I know people find it confusing to think about inversions, so can I use that somehow to make the game less obvious for you? Clearly, the anwer is yes &mdash; here's one way to do it:

As always, I show you three doors:

[ TODO the three doors doodle again ]

Just like before, I have you pick a door &mdash; and tell you, don't open it yet! Once you've picked your door, I hide a prize behind one of the doors, and then I give you a choice: would you like to *invert* your prize? 

What I mean is this:

If you choose to invert your prize, then when you open the door, one of two things will happen:

* If the door has a prize behind it, I keep the prize and you go home with nothing (cue Nelson Muntz laugh)
* If you pick one of the other doors, you get the prize. Hooray for you!

Alternately, you could choose *not* to invert your prize. That means, if you pick the prize door you get the prize, and if you pick a different door you get nothing.

You can see the game now hinges on an inversion, and so we shouldn't be surprised it's now kind of confusing. But you can think through it. How does your choice to invert affect your chances of winning?

Well, if you play the "inverted" game, then you're really trying to pick an *empty* door, because if you pick an empty door you get the prize. You have a 2 in 3 chance of picking an empty door, so you also have a 2 in 3 chance of winning. Alternately, you can choose not to invert, which means you have to pick the prize door to win the game: only a 1 in 3 chance.

If this sounds familiar, it should! Your choice to invert in *this* game is exactly like your choice of number of prizes in the older game. If you choose to play inverted winnings, it's like choosing to play with two prize doors; if you decide not to invert, it's like playing with just one prize door. You should choose to invert &mdash; it doubles your odds.

[ TODO the three doors doodle with the doors open and only one star prize, and some arrows showing you do get the prize for two empty doors but not for the one prize door ]

Make sure you understand how this inversion step affects your chances of winning before moving on. This is actually the key to the whole puzzle!

## Game 3: Monty Hall

You're reading on, so I guess you figured out game 2. It must not have been tricky enough after all! Time to make it even more confusing! I'm going to keep the inversion step, but now I'm going to obfuscate it so it's not clear there's an inversion at all. Ready? Watch and learn ...

First, I show you (you guessed it) three doors:

[ TODO the three doors doodle again ]

Once again, you pick a door &mdash; and once again, you don't open it just yet. Now, a new gimmick: I open one of the doors you didn't pick, and show you there was no prize behind it:

[ TODO three doors, one empty door doodle again ]

Now I ask: do you want to switch doors?

If it seems like switching doors shouldn't matter, look at the situation more carefully! There are only two closed doors left: a prize door, and an empty door. You have already picked a door. The door you already picked could be a prize door, or it could be empty. If you switch doors, you are guaranteed to land on the opposite kind of door.

That's right &mdash; switching doors inverts your prizes! Sounds familiar ... 🤔

In the second game variant, we decided we wanted to invert prizes because it turns our 2 in 3 odds of losing into 2 in 3 odds of *winning*. In this third game variant &mdash; the real Monty Hall game &mdash; the choice to invert has been hidden behind the guise of "switching doors." But there are always exactly two doors, and switching always inverts your priz, so your odds invert with it. You want the 2 in 3 odds of winning, so you choose to invert, so you choose to switch doors. Boom!

If you can work out the link between switching in this variant, and inverting in the previous game variant, then all of a sudden this once-confusing answer will seem pretty simple.
