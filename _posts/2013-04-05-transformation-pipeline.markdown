---
layout: post
title: The Transformation Pipeline
author: Dave
draft: true
---

If you've been playing with a real-time graphics package (like DirectX or
OpenGL) for a while, you've probably noticed how important transformations are.
They're used for several different purposes, like moving objects and 
positioning the viewer (i.e. the 'camera'). 

Things can get confusing when you start using the same tool for different
tasks! To deal with this, graphics developers over the years have converged
on a fairly simple system for naming different types of transformations.
This post will be an overview of vector spaces and transformations: what they
are, what they're called, and how they fit together into a pipeline. While it
seems everyone has their own naming system for spaces and transforms, the
basic ideas carry over universally.

We'll start with vector spaces. 

## Vector Spaces

A vector space is just a meaning given to the coordinate axis. This meaning is
chosen by convention. For example, we might define one vector space by saying
the $X$ and $Z$ axes align with the ground plane, with $Y$ pointing toward the
sky. We might define another vector space with $X$ and $Y$ being parallel to a
camera's lens, with $Z$ pointing out of that lens.

Vector spaces work on vectors a lot like how units work on numbers. For
example, the equation $5 + 2 = 7$ ...

* is correct when all the terms have the same unit: $5m + 2m = 7m$
* makes no sense when the units don't match: $5m + 2ft = 7(...?)$

For an equation with scalar numbers to make sense, the units attached to these
numbers must match. If you have different units on your terms, you can convert
some terms to a different unit of measurement. Once you're done converting,
the equation will make sense again:

$5 m + 2 ft * \dfrac{0.3048 m}{ft} = 5 m + 0.6096 m = 5.6096 m$

Vector spaces do more or less the same thing with vectors. If you have a vector
equation (with vector additions, dot products, or what-have-you), the math only
works if every vector is in the same vector space.
Vector transformations allow you to convert a vector from one vector space to
another. To continue with our analogy, these transformations are like unit
conversions. 

You might ask, then, how to transform vectors between vector spaces ...

## Transformations

You've probably seen vector transformations before. They allow you to 
scale, rotate and translate the objects in your scene.

Mathematically, each transformation is represented by a matrix. Say you have a
vector in the vector space $A$ ("$A$-space" for short), and you'd like to
transform it to $B$-space. If you already have the transformation matrix 
$M_{AB}$, you can simply multiply it with the $A$-space vector. The
result is the $B$-space vector. So, $v_B = M_{AB} \times v_A$.

Often $M_{AB}$ is simple. For example, if $B$-space is the same as $A$-space,
$M_{AB}$ is the identity matrix. If the origins of both spaces are the same,
but one unit in $A$ space equals two units in $B$ space, then $M_{AB}$ is a
scale matrix.
You can build complex $M_{AB}$'s by daisy-chaining the simpler ones.
They are combined via matrix multiplication:

* If $M\_{AB}$ transforms from $A$-space to $B$-space
* And $M\_{BC}$ transforms from $B$-space to $C$-space
* Then $M\_{AC} = M_{BC} \times M_{AB}$ transforms from $A$-space to $C$-space.

Intuitively, you might say $M_{AC}$ transforms from $A$-space to $B$-space to
$C$-space. 

If you have $M_{AB}$, which transforms from $A$-space to $B$-space, you can
obtain $M_{BA}$, which goes from $B$-space to $A$-space, by inverting $M_{AB}$.
If you had daisy-chained multiple transformations, you can obtain the inverse
by reversing the order of the steps and inverting each step. For example:

* If $M_{AC} = M_{BC} \times M_{AB}$
* Then $M_{CA} = M_{BA} \times M_{CB} = (M_{AB})^{-1} \times (M_{BC})^{-1}$

With this background out of the way, we can get a little more concrete.

## Transforming Objects

Scenes are composed of triangles. When you render your scene, you're sending a
lump of different triangles to the graphics system. While the graphics system just
sees a bunch of triangles in the end, we humans prefer to subdivide these triangles into
logical things. 'These triangles form a beach ball,' we might say, 'while these
triangles form a swing set.' 

Each one of these logical things is called an object. Graphics developers,
being human, like to talk about objects instead of triangles, but keep in mind an
object is just a bunch of triangles. Each triangle in an object is composed of
three vertices. Each vertex has a position, which is a vector in some vector
space.

When we say an object is in $A$-space, we really mean the position of
each vertex in the object is defined in $A$-space. When we say we transform an
object form $A$-space to $B$-space, we mean we transform the position of
every vertex in the object from $A$-space to $B$-space.

A natural next question is: how do we transform objects?

## The Transformation Pipeline

We mentioned earlier that there are a few different things we use
transformations for. There is a standard pipeline for doing all of these steps
in a way that won't make you want to pull your hair out (lucky, huh?). We'll
call this system the transformation pipeline. This pipeline is wholly separate
from the rendering pipeline!

Here's a rough outline of the transformation pipeline:

1. Start with a bunch of objects (a chair, a table, etc)
2. Transform each object so that it appears in the right part of the scene
3. Transform all objects in the scene as they would be seen from the point of
   view of the camera
4. Transform the view, so that things closer to the camera seem bigger than
   things farther from the camera
5. Now that vertices have been moved so they reside somewhere on the screen,
   render the scene by connecting vertices together and filling in triangles.

Each of the transformation steps has a common name, as do the vector
spaces they transform between. All that's left to do now is list them!

## Vector Spaces in the Transformation Pipeline

There are five standard vector spaces in the pipeline. In theory, you can
define these spaces however you like. Where possible, we'll give the concrete
definitions that OpenGL uses.

### Object Space

We mentioned that the positions of all the vertices in an object reside in some
vector space. Object space, then, is just a convenient coordinate system for 
defining those positions.
Notice we said _an_ object: each object in your scene will likely
have its own object space.

Consider that hypothetical beach ball from before. We might define the object 
space for that beach ball like this:

* The origin of the beach ball's object space coincides with the center of the
  ball
* The $Y$ axis aligns with the north and south poles of the beach ball
* The unit length along any axis is the same as the radius of the beach ball

These choices were arbitrary. For example, we might instead have decided to
make the origin of our object space coincide with the north pole. Or the south
pole. Or a random point on the ball. Or anywhere else outside the ball. We
chose the space above just to make it convenient to specify the position of
each vertex in the ball.

### World Space

We mentioned there were multiple object spaces (one for each object). To
compose objects into the same scene, it's useful to think of a scene-level
vector space in which objects can be placed together. This space is called 
world space.

World space is typically defined so the center of the scene is at the origin.
Sometimes the unit length in world space corresponds to physical length. This
helps the scene creator reason about world-space coordinates.

### Eye Space

(also known as camera space, or view space)

The camera allows us to conceptualize
the viewer's position and orientation. Eye space is simply the camera's object
space. In OpenGL, it is defined so that the camera is at the origin, looking 
down the negative $Z$ axis, with $Y$ pointing up.
As we'll see later, eye space is important for rendering. Although scenes are
easiest set up in world space, it is easiest to render the scene after
transforming all the objects in the scene to eye space.
Transformations to the rescue!

### Clip Space

Clip space defines vertices relative to where they appear on your screen. In
OpenGL ...

* the origin of clip space is defined as the center of your monitor. 
* $X = -1$ and $X = 1$ correspond to the left and right boundaries of your
  monitor
* $Y = 1$ and $Y = -1$ correspond to the top and bottom of your monitor
* The $Z$ component determines visibility when triangles overlap. A vertex with
  $Z = 0$ is in front of everything; a vertex with $Z = -1$ is behind everything.

Anything outside the bounds listed above will not be rendered. The graphics
pipeline will remove geometry outside these bounds, in a process called
_clipping_ (hence the name _clip space_).

### Screen Space

Screen space is defined to make it easy to work with pixels. It's a 2D space,
with no $Z$ coordinate.

Let $w$ and $h$ be with width and height of your display, in pixels. In screen
space,

* $(0, 0)$ corresponds to the top-left corner of your display
* $(w-1, 0)$ corresponds to the top-right
* $(0, h-1)$ corresponds to the bottom-left
* $(w-1, h-1)$ corresponds to the bottom-right

Thus every 2D coordinate $(i, j)$ is the index of a pixel. This is why
screen-space coordinates are sometimes called pixel coordinates.

## Fitting These Together

To recap, we listed five spaces above:

* __Object space__: convenient for defining vertex positions in an object
* __World space__: convenient for setting up those objects into one scene
* __Eye space__: the same scene, but from the camera's point of view
* __Clip space__: the final space containing what will be rendered
* __Screen space__: clip space, but engineered for cloring pixels

A typical workflow using these spaces might work as follows:

1. An artist creates a mesh, defined in some object space that was convenient
   for the artist when he created the mesh.
2. A graphics programmer writes code to transform each object into world space,
   creating a coherent scene.
3. The graphics programmer writes code to let the user manipulate their
   viewpoint, and uses that input to transform the scene to eye space.
4. The graphics programmer uses a transformation to foreshorten the scene,
   making nearby objects seem closer. This results in clip-space coordinates.
5. The graphics programmer sends the object vertices and transformations to the
   graphics system.
6. The graphics system renders these vertices on the screen, in screen space.

## Matrices in the Transformation Pipeline

Now that we've finished naming spaces, it's time to name the transformation matrices.
There are four transformation steps in our pipeline, but only three of them are
implemented using matrices. 

### Object $\rightarrow$ World

Called the *world* or *model* matrix. A composition of transformations that
define the position, size and orientation of each object in the scene.
Since there is one object space for each object, there is also one world matrix
for each object. 

### World $\rightarrow$ Eye

Called the *view* matrix. This matrix is used to translate and rotate each
object in the scene so that the entire scene is defined in eye space, relative
to the camera. This is the opposite of the sequence of steps that place the
camera into the scene (think back to matrix inversions).

### Eye $\rightarrow$ Clip

Called the *projection* matrix. This step usually includes perspectivization,
which makes closer objects seem bigger. This simulates visual perspective.

### Clip $\rightarrow$ Screen

This does not have a name, because it's usually handled by the graphics system
instead of the programmer. The graphics system implements this step by dropping
the $Z$ term and scaling the $X$ and $Y$ terms by the width and height of the
screen, in pixels.

## Implementing the Transformation Pipeline

That's the transformation pipeline! It goes by many names, its steps are
referred to by different names, and sometimes different steps are grouped in
weird ways; however, the core pipeline is still pretty much the same in every
instance. It's always the same sequence of transformation steps.

Let's close out by thinking how we could use this transformation pipeline to
render the scene. Here's a simple algorithm:

    // Start in object space
    orig := load_objects()

    // object -> world space
    objects := empty list
    for i in (0 -> orig.length):
        objects[i] = orig[i].doWorldMatrix();

    // world -> view space
    for i in (0 -> objects.length):
        objects[i] := camera.doViewMatrix(objects[i])
    
    // view -> clip space
    for i in (0 -> objects.length):
        objects[i] := camera.doProjectionMatrix(objects[i])

    // Let the graphics system do clip -> screen space and render the scene
    for i in (0 -> objects.length):
        for j in (0 -> objects[i].numTriangles):
            graphicsSystem.drawTriangle(object[i].triangle[j])

No graphics system (or at least none known to the author) is set up to work
this way. This is because you can save space by swapping the two loops. That
is, the above snippet basically does this:

    for each transformation step
        for each object
            transform the object

Whereas we can save memory by doing this:

    for each object
        for each transformation step
            transform the object

We save space by not needing to store the intermediate object vertices. The
resulting snippet looks something like this:

    // Start in object space
    objects := load_objects()

    // Render the objects
    for i in (0 -> objects.length):
        objectToClip := identityMatrix()
        objectToClip *= objects[i].getWorldMatrix()
        objectToClip *= camera[i].getViewMatrix()
        objectToClip *= camera[i].getProjectionMatrix()

        for j in (0 -> objects[i].numTriangles);
            graphicsSystem.transformAndDrawTriangle(
                object[i].triangle[j],
                objectToClip
            )

This is the basic 'accepted' implementation of the transformation pipeline.
Each graphics pipeline has its own separate twist on this pipeline, but the
idea remains the same.

## Where to Next?

If you're developing shaders, you may be interested in the OpenGL-based
followup article, [Shader Transformations](/04/05/shader-transforms.html),
which builds on what you learned in this article.
Otherwise, onward to your next graphics programming adventure! :D

