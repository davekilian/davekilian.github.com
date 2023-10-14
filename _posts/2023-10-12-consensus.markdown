---
layout: post
title: (Unified Consensus Article)
author: Dave
draft: true
---

If you ask some rando off the street, "Hey, what are some foundational problems in the field of distributed systems?" they'd probably say something like, "What? Who are you? Get away from me!" Others might suggest the problem of *distributed consensus* &mdash; the problem you solve with fancy algorithms like Raft and Paxos.

If you work on, in or near a distributed system, you'll run into the problem of consensus sooner or later; if you work on distributed systems code, the way you think about solving consensus is the way you think about solving quite a few day-to-day problems. Besides, if you run a distributed system, you have a consensus algorithm running somewhere in your infrastructure. Having a basic grip on consensus algorithms is incredibly useful, even if you never need to implement one yourself.

It's unfortunate, then, that consensus algorithms are hard to explain &mdash; Paxos notoriously so. I'm going to try a different approach. I believe the usual "boil it down" approach falls apart with Paxos, because at the end, you're still left with many moving pieces working together in subtle ways. A better way to do it is loosely trace the line of thinking that lead to the algorithm's original invention. Is this actually better? You decide!

Slow and steady wins the race! Let's start with a little example:

## A Story About Consensus

You're a bright, young, enterprising software developer, ready to start your journey in the land of distributed systems; so optimistic in your soon-to-be-newfound abilities, you go to Github and register the account `paxosexpert`.

At least, you think you do.

But the first time you log into your account, you get a Jedi Octocat with a 404 error. (You think a *The Matrix* "There is no spoon" reference would be more apt movie reference for a 404-not-found page, but you also accept the world you live in is imperfect.) You refresh once, twice, but your account doesn't load. Frustrating. You go back to the account creation page, and create your account. Or, at least you try to &mdash; but you get an 'account already exists' error. Hmm. You refresh and try creating your account a third time. It works! You log in!

You deck out your profile, upload your best profile pic, you mash that big green Create Repository button start it off with an awesome readme filled with in-jokes, you get that ... 404 page? No ... You hit refresh. You're back in! But everything you did is gone! You're back to a blank account. All gone &mdash; it takes a moment to sink in. Well, time to pick up the pieces. Once again, you fill out your profile, you create that repo, you throw together that readme. Maybe a bit more rushed than before, maybe you took a small hit to that initial rush of enthusiasm, but your account's all set and you're happy with it. You send a link to your new account to your mom. (You two have a great relationship.)

Your phone buzzes: *A Star Wars cat! So funny!"

Star Wars cat ... the 404 page again?! *No!* You open your GitHub profile again in a hurry. Relief! It's not a 404 page! It's your profile, it's ... wait, it's, your first profile? The one that disappeared before &mdash; the one you built slowly, painstakingly, with love, not the one you rushed through recreating. You're not mad, but you're not happy either. More like baffled. What's going on?

There's actually a simple explanation ...

###  A Tale of Two Servers

This is what was happening behind the scenes:

Like in real life, the fictional GitHub in this story is a distributed system of servers running the cloud. When you connect to GitHub, you can end up connecting to any of these servers. In this story, you were talking to two; let's call them *Morpheus* and *Neo* (or $M$ and $N$ respectively, for the mathematically inclined). Behind the scenes, *Morpheus* has been having a very bad day &mdash; its network connection is flaky, so it keeps going offline, online, offline, online randomly all the time. *Neo* is working just fine, but is out of sync with *Morpheus* (doesn't help that *Morpheus* can't seem to stay up).

Let's see what was happening behind the scenes:

*"[...]  you go to Github and register the account `paxosexpert`."* The first time you create your account, you get hooked up with *Morpheus*. It creates your account, saves your account details and sends you to the login page.

*"But the first time you log into your account, you get a Jedi Octocat with a 404 error."*. Looks like *Morpheus* went down; you got automatically redirected to *Neo*, which is working, but doesn't know about your account yet. Hence the error.

*"You go back to the account creation page, and create your account. Or, at least you try to &mdash; but you get an 'account already exists' error."* You're back with *Morpheus* again. It knows your account exists, so it won't let you create it a second time.

*"You refresh and try creating your account a third time. It works! You log in!"* And, just like that, *Morpheus* is down again and you're back with *Neo*! *Neo* still hasn't heard about your account from *Morpheus* so it lets you create your new account a second time. This is really bad! Now there are two copies of the same GitHub account floating around!

And that's why there appear to be two diverged copies of the account floating around throughout the rest of the story. Why did Mom see a 404 error? Maybe she got routed to some other server that hadn't heard of either copy of your account yet. (*Trinity*?)

So, what did we learn here? What's wrong with this system? Why doesn't this happen in the real-life GitHub? In real life, the real GitHub servers avoid these problems by staying in sync; the problem of staying in sync is what we mean by *consensus*. A consensus algorithm is one these servers can run so that, at all times, all servers agree whether `paxosexpert` is an account somebody already created, or an unused account name available to be registered, even if the account was registered by flaky ol' *Morpheus*.

###  Out of Many, One

Our little story about flaky websites and flaky servers also exemplifies something important about distributed systems: the goal of any distributed system is to take a bunch of computers and make them seem to a user like one really big computer. That means, anything the user knows, all servers must know and never forget. To make a pile of servers into a distributed system, you have to keep them in sync &mdash; you need consensus!

That's why people talk about consensus as being "foundational" to distributed systems. It's also why I could be so confident at the very start of this document in saying, if you have a distributed system, then something in your system is running a consensus algorithm. Simply put, if your distributed system doesn't solve consensus somewhere, nothing can stay in sync, and so you don't actually have a distributed system.

Okay, enough navel-gazing. Consensus is important. So how does one solve it?

They say the journey of a thousand miles begins with a single step, but before that single step, you need to decide to go on that journey in the first place. Just so, before we can begin designing consensus algorithms, we have to decide what they should do first.

##  Properties of a Consensus Algorithm

TODO the short thing is to start with it being more complicated than just replication. Consensus implies the potential for conflicting opinions and disagreement that has been overcome or avoided; a consensus algorithm is really a fault-tolerance conflict-resolution algorithm. Then we can unpack the properties out of htat.

TODO

* Coherence - the foundational property, everyone eventually in sync
* Conflict-Resolution - even if two conflicting values appear concurrently
* No-decoherence - as soon as you can declare the algorithm done, it is impossible to forget or change your mind
* Fault Tolerance - even if a reasonable number of machines die (for some def of reasonable) ...
  * Safety - the other properties above are still maintained
  * Liveness - the algorithm does not get stuck

