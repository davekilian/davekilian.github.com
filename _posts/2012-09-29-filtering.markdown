---
layout: post
title: An Image Filtering Overview
author: Dave
draft: true
---

Every year, [CS123](http://cs.brown.edu/courses/cs123) students spend a couple weeks
learning about image filtering in class before going off on their own to 
implement some basic filters, blurs and scaling algorithms. Filtering, like
signal processing in general, can be tricky. It's easy to get lost in the
theory when you start talking about image sampling and reconstruction.

So, let's see if we can put everything in perspective.

## Duals

Before we can dive into signal processing, we're going to need a theoretical
tool to help us talk about our signals. That tool is the _dual_. 

Think back to calculus. Consider a function $f(x)$, and its derivative 
$f'(x)$. 

    TODO graphs of a function and its derivative

When you take the derivative of a function, you get back another function.
This derivative says something about the original function, but may look
nothing like the original function at all. You can integrate the derivative to
get back the original function.

If you change $f(x)$, you get back a different $f'(x)$. If you modify
$f'(x)$, integrating gives you a different $f(x)$ than the original.
The function and derivative seem to be tied to one another.
You can think of them as _duals_ of each other.

    TODO graphs showing ramifications of modifications

Disclaimer: mathematically-inclined readers are probably shaking their
heads in disgust right now :) While this example is a useful way to gain
intuition about duality, the real definition is a bit stronger: duality
requires the same operation to transform something into its dual and back.
Derivation and integration, however, are entirely different beasts. 

## The Frequency Domain

So why are we talking about duals? Because we'll find one duality particularly
useful for image processing: the duality between the spatial and frequency 
domains. The spatial domain is another name for the types of functions you've
been graphing all your life. But what is the frequency domain?

It turns out every function can be represented as a sum of infinitely many 
sine waves. Specifically, if we took one sine wave for every possible 
frequency $f$, multiplied each by a different amplitude $a\_f$, and summed 
all of those, we get back some function. By carefully choosing our values
of $a\_f$, we can exactly recreate any function this way. 

In other words, every function $g(x)$ can be rewritten as follows:

$$g(x) = \sum\_f a\_f \sin(fx)$$

### Visualizing the Frequency Domain

Like regular (spatial domain) functions, frequency domain functions can be
graphed. However, the axes of frequency graphs mean something 
different than the axes of spatial domain graphs. 

In the spatial domain, the graph axes represent spatial units. The graph
is actually just a 2D space, in which a function is drawn. For frequency
graphs, however,

* The horizontal axis represents **frequency**
* The vertical axis represents **amplitude**

Then each point on the frequency graph specifies the frequency and amplitude
of one of the infinitely many sine waves.

Let's try some examples. Consider $g(x)$, a spatial function consisting of a 
simple sine wave:

    TODO graph of a simple sine wave

The frequency graph of $g(x)$ looks like this:

    TODO line at y = 0, single point at (f, a_f)

Notice the horizontal line at $y = 0$. The amplitude of every sine wave,
except one, is 0. So none of these sine waves contributes to the sum
that produces $g(x)$. 

However, the point at $(f, a\_f)$ specifies the sine wave with frequency
$f$ has amplitude $a\_f$. So if we transform this frequency
graph back into the spatial domain, we get back the original $g(x)$:

$$g(x) = \sum\_i a\_i \sin(ix) = 0 + ... + a\_f \sin(fx) + ... + 0 = a\_f \sin(fx)$$

In principle, we can do this for any function $g(x)$, even if $g(x)$ isn't
a trigonometric function (e.g. $g(x) = x^2$). For now, you should treat
the transformation that does this as a black box. If you're interested later,
you can read more about the transformation, which is called a
[Fourier transform](http://google.com/search?q=Fourier+transform).

Whew! With that bit of theory out of the way, we can start talking about the
images themselves...

## Representing Images Mathematically

From here on in, we'll only talk about 1D images (i.e. those consisting of a
single row of pixels), in grayscale. Everything we talk about will generalize
to 2D color images, however. This just makes the graphs easier to draw :)

For analysis, we're going to represent an image using a function. This
function is defined over the spatial extents of the image. For every point 
$(x, y)$ on the graph, $y \in \[0, 1\]$ denotes how much light is measured
or displayed at point $x$ in the image. The value $y = 0$ denotes pure black 
and $y = 1$ denotes pure white. All values in between are shades of gray.

Below is a 1D image and its corresponding image function:

    TODO example graph and image

This function is called the **signal**. The term originates from **signal
processing**, the field where most of this content originated.

## Sampling

On a computer, we represent the signal by **sampling** it.
To sample the signal, we move along the $x$ axis, stopping at regular intervals
and recording the $y$ value at each $x$. The result is a set of $(x, y)$ points
from the original function. We call these points **samples** or **pixels**.

    TODO diagram for sampling the graph above

Note that, since samples are points, they have no associated width or height.
You may be used to thinking of pixels as little squares, but for this article
you should think of pixels as infinitesimally small points, with zero width
and height.

Of course, we lose information when we do this. Given only a set of sample 
points, we can only guess at what the original function looked like. For
example, given these points:

    TODO points

The original function could have looked like this:

    TODO lerped

Or this:

    TODO cubic interpolation

Or even this:

    TODO something crazy

The process of recreating a function from its sample points is called
**reconstruction**. Reconstruction isn't perfect, but it works well enough
given enough input data. More input data makes reconstruction more accurate.

## The Nyquist Limit

We mentioned that we sample by moving along the $x$ axis at regular intervals.
One natural question is: How big should these intervals be? Using a smaller 
interval gives us more data (thus improving reconstruction), but it requires
more storage space. How large can we get away with making our sampling
interval without making our reconstruction terrible?

Harry Nyquist approached this problem by transforming the signal into the
frequency domain. His Nyquist Limit works as follows:

>   Look at the frequency domain representation of the image. Find the 
>   highest-frequency point that has a non-zero amplitude. Let $f$ be the
>   frequency associated with this point. $f$ is the highest frequency
>   component of the continuous image signal.
>   
>   &nbsp;
>   
>   The wavelength of $f$ is $\frac{1}{f}$. To faithfully represent the 
>   original function, the sampling interval must be no larger than 
>   $\frac{1}{2f}$. Equivalently, the sampling frequency must be $\geq 2f$. 

To gain insight on why this is true, try playing with
[this CS123 applet](http://www.cs.brown.edu/exploratories/freeSoftware/repository/edu/brown/cs/exploratories/applets/nyquist/nyquist_limit_java_browser.html).

## Dealing With the Nyquist Limit

Thanks to the Nyquist limit, we now know how to sample an image: just choose a
high enough sampling frequency to faithfully represent the image, and sample
the image intensity at each of the sample points.

There's a hitch: the maximum frequency of an image depends on what you point
your camera at, but your camera's sampling rate is fixed. Digital cameras
sample using a rectangular array of CCDs, each of which samples the light
intensity at that point. The camera's sampling rate is constant, determined 
by how far apart the CCDs are placed.

    TODO diagram of a signal with high frequency stuff

Although we cannot faithfully represent signals from scenes with high
frequency data, we can represent a subset of the scene. What we need is a
modified version of the scene, identical to the original except with all of
the high frequency data somehow cut out.

    TODO diagrams of corresponding filtered signal

This, finally, is where the duals (and, indeed, filtering in general) come 
into play.

## Low-Pass Filtering

The filter that removes high-frequency data is called a low-pass filter. The
name simply means _low_-frequency data is _pass_ed through the filter, while
high frequency data is removed. As you might imagine, this is where the
frequency domain comes in handy!

Consider our original signal from a little while ago:

    TODO spatial graph

Here's its frequency graph, determined by a discrete Fourier transform:

    TODO frequency graph

To filter out the high frequencies, we just zero the amplitude at any point
beyond the maximum allowable frequency $f\_{max}$ (determined by the
Nyquist limit):

    TODO another frequency graph

Then we can reconstruct the spatial signal using a Fourier transform:

    TODO filtered graph

Now we can sample with impunity!

---

TODO

* Box function

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

