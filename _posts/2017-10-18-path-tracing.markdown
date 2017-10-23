---
layout: post
title: Path Tracing
author: Dave
draft: true
---

Brain dump:

Assume the reader can write a basic raytracer. Many tutorials online otherwise, I’m sure we can link to a couple

Start with motivation.

Basic raytracer, maybe using G3D explicitly as boilerplate.

Physics of light. Photons are particles and waves, but that’s not as crazy as it sounds, they’re basically transverse waves that travel in straight lines like a particle would. Like sound is waves of high pressure followed by low pressure.

Physics of reflection: photon strikes an electron and excites it, after some time the photon will be emitted back out and the electron will return to a lower energy state. Some randomness in this process, so we can’t say exactly which direction the electron will be traveling when the photon is emitted, or the direction the photon will be going.

But we can measure aggregate. Introduce the idea of a BRDF, it’s not as fancy as the name makes it sound. You can measure a BRDF empirically using a gonioreflectometer. You can use phong as a BRDF if you like.

Let’s change our raytracer to use a BRDF factored out of the tracer itself. We can move phong into the BRDF routine. Critically, we can replace phong with anything else we like.

Now let’s try to answer a different question: how much light is leaving a particular point in a particular direction. Notice that if we could answer this question, we can build a renderer: for each point under a pixel in the scene, compute the light leaving that point in the direction of the camera.

Introduce kajiya rendering equation as a simple model. It’s really easy, it just falls out of the definition of a BRDF and the physics we already talked about.

The problem with this equation is solving it. There are infinitely many directions to integrate over, and in each direction we need to integrate an infinitely deep recursion. Two infinities, send help!

We don’t need to solve the integrals here to be successful, we just need to evaluate them. And given we’re rendering for a digital display for imperfect human eyes, we don’t even need a perfect evaluation, an estimate will do.

We’ll use a trick called Monte Carlo Estimation. Talk about history. Talk about Monte Carlo vs Las Vegas

Monte Carlo is basically just a trick you can do with Riemann sums. Show if you have a Riemann sum with N boxes, you can evaluate one more point to obtain a Riemann sum with N+1 boxes. A Riemann sum with 1 box is a terrible estimate, but once you have a couple hundred things actually start to look pretty good. That’s the heart of MCE, the idea is just to evaluate the integral at random points and divide out by the number of points you evaluated out.

We’re not out of the woods yet though. Before we had infinitely many infinitely recursive integrals to evaluate. Now we’ve reduced that to a finite number of infinitely recursive integrals.

Russian Roulette. Basically start by saying, well, what if we just stop evaluating after a little while? Maybe a 10% chance of ending the recursion at each level. Unfortunately this “biases” the end result.

If you have a 90% chance of returning f(x) and a 10% chance of returning 0, on average your evaluation comes out at .9f(x)...which is not f(x). Solution: just divide by .9

With .9 probability, we return f(x)/.9 and with .1 probability we return 0. On average that’s f(x)

Now we’re finally at a tractable problem: evaluate finitely many finitely recursive integrals. Seems like we’re finally ready to write a little code.

Then derive the example path tracer.

Include a final example snippet and some renders.
