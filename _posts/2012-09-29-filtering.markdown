---
layout: post
title: Understanding Image Filtering
author: Dave
draft: true
---

Every year, [CS123](http://cs.brown.edu/cs123) students spend a couple weeks
learning about image filtering in class, before going off on their own to 
implement some basic filters, blurs and scaling algorithms. Filtering (and
signal processing in general!) can be tricky. When it comes to image sampling
and reconstruction, it's easy to get lost in the theory.

Let's see if we can put everything in perspective.

## Duals

Before we can dive into signal processing, we're going to need a theoretical
tool to talk about our signals. So let's talk about _duals_. 

Think back to calculus. Consider a function _f(x)_, and its derivative 
_f'(x)_. 

    TODO graphs of a function and its derivative

When you take the derivative of a function, you get back another function.
This derivative says something about the original function, but probably looks
nothing like the original function at all. You can integrate the derivative to
reconstruct the original function.

If you change the function, you get a different derivative. If you modify
the derivative, you integrate to a different function than the original.
You can think of the function and derivative as _duals_ of each other.

    TODO graphs showing ramifications of modifications

Disclaimer: mathematically-inclined readers are probably shaking their
heads in disgust right now :) While this example is a useful way to initially
understand duality, true duality is actually a bit stronger than this. It
requires using the same operation to transform into the dual and back.
Derivation and integration, however, are entirely different beasts. This 
wasn't a true example of duality.

## The Frequency Domain

So why are we talking about duals? Because we'll find one duality particularly
useful for image processing: the duality between the spatial and frequency 
domains. What's that?

It turns out every function can be represented as a sum of infinitely many 
sine waves. Specifically, if we took one sine wave for every possible 
frequency, multiplied each by a different amplitude, and summed all of those, 
we could get back any function by choosing our amplitudes carefully. 
This isn't totally obvious!

---

TODO

Spatial / Frequency Dual
* Example with a simple sine wave. Be sure to explain how the horizontal axes don't
  line up between the duals.
* More complex example
* This is a theoretical tool. We'll use it in a minute ...

Sampling
* Images as continuous functions
* Our representation is a discretization -- sampling
* What information can we represent with sampling? Nyquist limit
* What happens when we don't adhere to the Nyquist limit?
* So need to filter at twice the frequency of the signal from the real world.
* But the number of samples we take is fixed ...
* Compromise: change the signal, throwing out any high frequency noise
* This is called low-pass filtering

Low-Pass Filtering
* Why did we take this digression? 
* Because low-pass filtering is really easy in the frequency domain. Just multiply
  by the box function!
* Show that it works.

Perspective!
* To recap, we're trying to represent images using a limited number of discrete samples.
* To do this, we'll filter out high frequency noise. The result is a similar signal, but contains less information.
* Here's our new algorithm:

    Use FFT to create the frequency domain equivalent of the signal
    Zero the high frequencies we can't represent
    Use InvFFT to reconstruct the modified spatial domain function
    That's our image!

There's a Better (Read: Faster) Way
* All this is complex math. It's probably pretty complicated and slow. Let's not deal with it?
* It turns out you can do something completely different, entirely in the sample domain, to get the same result.
* Namely, we can do _convolution_ using the _sinc(x)_ function. 
* Show sinc.

Convolution
* Urgh. Hard to explain. 
* I'll deal with this later haha

Actually, That Strategy Sucks Too
* I have a few stock answers for why it sucks, but I can't explain why (i.e. why are negatives really that bad?)
* So look it up and have a solid answer for this

Approximate!
* We want functions with the same general shape
* The box is a kind of silly approximation, but even then it's a first-order approximation.
* Triangle is better
* Discrete gaussians are pretty close.

Recap
* What we just did was low-pass filtering, "pass" (preserve) lower frequencies while deleting high frequencies.
* We did this so we can preserve just the information we can actually represent with our pixel grid
* There are more complexities, of course (taking a picture implicitly does some filtering). Read hughes/van dam (or CS123 slides!).
* There's a related problem ...

---

This should probably be a different but shorter blog post

Image Scaling
* Idea that we're taking more or less fine-grained regularly distributed samples.
* Let's look at scaling up first

Reconstruction
* We're missing data. 
* All we have is these points. What happened in between?
* Some examples of what could have happened
* So we'll just guess. What's a good guess?
* Some different interpolation functions. 
* We can just sample along these graphs. Noice!

Downscaling
* We can use the same strategy for scaling images down, right?
* One major problem ... what happens to the frequencies when we scale down?
* Oh no. They go up. What if we have high frequencies again?
* Well we know what to do. Filter them out first! Then resample.
* It turns out you can filter and intepolate in one step, using a single filter function.

Recap
* Again we passed over stuff. 
* Hopefully you at least have a good enough intuition to talk about theoretical complexities now.
* Happy filtering!

