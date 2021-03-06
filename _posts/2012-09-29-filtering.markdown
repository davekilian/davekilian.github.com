---
layout: post
title: A Sampling Overview
author: Dave
draft: true
---

Every year, [CS123](http://cs.brown.edu/courses/cs123) students spend a 
couple weeks learning about image filtering in class before going off on 
their own to implement some basic filters, blurs and scaling algorithms. 
Signal processing can be tricky. It's easy to get lost in the
theory when you start talking about image sampling and reconstruction.

So, let's see if we can put everything in perspective.

## Duals

Before we can dive into signal processing, we're going to need a theoretical
tool to help us talk about signals. That tool is the _dual_. 

Think back to calculus. Consider a function $f(x)$, and its derivative 
$f'(x)$. 

    TODO graphs of a function and its derivative

When you take the derivative of a function, you get back another function.
This derivative means something about the original function, but may look
nothing like the original function at all. You can integrate the derivative to
get back the original function.

If you change $f(x)$, you get back a different $f'(x)$. If you modify
$f'(x)$, integrating gives you a different $f(x)$ than the original.
The function and derivative seem to be tied to one another.
You can think of them as _duals_ of each other.

    TODO graphs showing ramifications of modifications

Disclaimer: Although this example captures the main idea, it's a bit of an
oversimplification. The real definition of duality is a bit stronger: duality 
requires the same operation to transform someting into its dual and back. 
However, derivativation and integration are entirely different beasts.

## The Frequency Domain

So why are we talking about duals? Because we'll find one duality particularly
useful for image processing: the duality between the spatial and frequency 
domains. What are those?

It turns out every function can be represented as a sum of infinitely many 
sine waves. Specifically, if we took one sine wave for every possible 
frequency $f$, multiplied each by a different amplitude $a\_f$, and summed 
all of those, we get back some function. By carefully choosing our values
of $a\_f$, we can exactly recreate any function this way. 

In other words, every function $g(x)$ can be rewritten as follows:

$$g(x) = \sum\_f a\_f \sin(fx)$$

Once this makes sense, we can define the difference between spatial and
frequency domains:

* The **spatial domain** is how you've represented functions all your life.
  It's also how we defined $g(x)$ above. In the spatial domain, functions
  take in an $X$ value and produce a $Y$ value. The $Y$ value denotes the
  height of the function above the axis at a given $X$ point.

* The **frequency domain** may be new. In the frequency domain, functions 
  take in a frequency and produce the amplitude of the sine wave with that 
  frequency. That is, the function takes an $f$ value and produces the
  corresponding $a\_f$ value.

## Visualizing the Frequency Domain

Like regular (spatial domain) functions, frequency domain functions can be
graphed. However, the axes of frequency graphs mean something different than 
the axes of spatial graphs. 

Remember spatial domain functions take in an $X$ and produce a $Y$. Similarly,
frequency domain functions take in an $f$ and produce an $a\_f$. To graph
spatial domain functions, we represent $X$ with a horizontal axis and $Y$
with a vertical axis.

Knowing that, how do you think we use the axes of a frequency graph?

The answer:

* The horizontal axis represents **frequency**
* The vertical axis represents **amplitude**

So each point on the frequency graph specifies the frequency and amplitude
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

For analysis, we'll use functions to represent images. These functions are
defined over the spatial extents of the image. For every point 
$(x, y)$ on the graph, $y \in \[0, 1\]$ denotes how much light is measured
or displayed at point $x$ in the image. The value $y = 0$ means pure black 
and $y = 1$ means pure white. All values in between are shades of gray.

Below is a 1D image and its corresponding image function:

    TODO example graph and image

This function is called the **signal**. The term originates from signal
processing, the field where most of this content originated.

## Sampling

On a computer, we represent the signal by **sampling** it.
To sample the signal, we move along the $x$ axis, stopping at regular intervals
and recording the $y$ value at each $x$. The result is a set of $(x, y)$ points
from the original function. We call these points **samples** or **pixels**.

    TODO diagram for sampling the graph above

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
**reconstruction**. Reconstruction isn't perfect, but it works fine
given enough input data. More input data makes reconstruction more accurate.

## The Nyquist Limit

We mentioned that we sample by moving along the $x$ axis at regular intervals.
One natural question is: How big should these intervals be? Using a smaller 
gives us a better reconstruction (because there's more data), but it requires
more storage space. How large can we get away with making our sampling
interval without ruining the reconstruction?

Harry Nyquist approached this problem by thinking in the frequency domain.
His Nyquist Limit works as follows:

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

There's a htich: the maximum frequency of an image is highly variable, but
your camera's sampling rate is fixed. Digital cameras
sample using a rectangular array of CCDs, each of which samples the light
intensity at that point. The camera's sampling rate is constant, determined 
by how far apart the CCDs are placed.

    TODO diagram of a signal with high frequency stuff

Although we cannot faithfully represent signals from scenes with high
frequency data, we can represent a subset of the scene. What we need is a
modified version of the scene, identical to the original except with all of
the high frequency data somehow cut out.

    TODO diagrams of corresponding filtered signal

This is where image filtering comes into play.

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

## Filtering Math

As it will turn out later, it's useful to represent the idea of filtering
in the frequency domain as multiplying the frequency function by a filter
function. That is,

$$result(x) = filter(x) \times signal(x)$$

For low-pass filtering, we want all frequencies from $-f\_{max}$ to $f\_{max}$
to keep the same amplitude. We want all frequencies outside that range to have
an amplitude of 0. The former is achieved by multiplying the amplitudes by 1;
the latter is achieved by multiplying by 0.

Let $box(m, x)$ be defined as follows:

    if |x| < m              
        return 1
    else                    
        return 0

$box(m, x)$ looks like this:

    TODO graph

To low-pass filter in the frequency domain, we can multiply the signal by
$box(f\_{max}, x)$:

$$result(x) = box(f\_{max}, x) \times signal(x)$$

    TODO graph of example multiplication

## Recap

We've gone off on a few tangents now, so maybe it's time to remind ourselves
what exactly we're trying to do.

We want to represent images using discrete samples. To do this correctly, there's
a minimum number of samples we must take. The Nyquist limit tells us exactly how
many.

Unfortunately, the resolutions of our cameras and monitors is fixed. 
Instead of taking more samples, we have to reduce the complexity of the image.

Here's our algorithm for doing that

* Use a Fourier transform to obtain the frequency plot of the signal
* Multiply by the box function to remove high frequencies we can't represent
* Use a Fourier transform to obtain the spatial plot of the modified frequency 
  plot
* That's our image!

## Why is Any of This Useful Anyway?

---

TODO

* Mention that the camera does all the prefiltering in hardware already.
* All this stuff is only useful when you need to resample an image for some reason, like scaling.
* Maybe want to restructure things a bit actually. Otherwise this section seems kinda random.

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

