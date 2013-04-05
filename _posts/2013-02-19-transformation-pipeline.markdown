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
from the rendering pipeline, which we'll get to near the end of this article.

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

## Putting It Together

---

## TODO

* Talk about how these transformations are baked together, and how this is a
  win from a Big-O standpoint. Start with some pseudocode for how you might
  do this naively by transforming objects in stages, then move on to the more
  efficient formulation the graphics pipeline uses today.

* Talk about how these transformations entirely encapsulate the notion of a
  camera. DirectX and OpenGL do not provide a camera abstraction because none
  is needed, although it is useful for you to create your own.

Everything else below is old. The OpenGL stuff should be split into a new post.
Then change the title and rewrite the intro so we don't talk about shaders at
all in this post.

---

## The old intro

Debugging shaders is a tricky business, especially when you can't use
printlines or a proper debugger. This makes preventing shader bugs a better 
strategy than trying to track them down and fix them.

This post is about preventing the most common shader bug: mixing up your 
transformations. Shaders with this type of bug usually look right (or almost
right) from one point of view, but do something weird as soon as things start
moving around. One of the most common symptoms is lights always moving relative
to the camera.

Preventing this bug requires a little discipline, but it's probably something
you've done before. Let's start with something a bit more familiar.

---

### Conclusion

This pipeline transforms triangle vertices from their original object-space
positions to final screen-space pixel positions. We can perform the entire
pipeline by composing the matrices from all the steps. Then we can render the
triangles as follows:

    Obtain scene global matrices (everything except the world matrix)

    For each object
        Obtain the object's world matrix
        Compose all the matrices above in the correct order

        For each triangle 
            Multiply the vertices by the final matrix
            Figure out which screen pixels the vertices enclose (rasterization)
            Figure out what colors to assign those pixels (shading)

In essence, this how GPUs render scenes.

---

## Idea

Split the rest of this post into a different post. The topic is similar but the
writing style is pretty different. Plus this post is super long.

---

## Transformation Matrices in OpenGL

The rest of this post will focus on using the OpenGL compatibility profile (with 
`glPushMatrix()` and friends) to write shaders.

### OpenGL's Matrix Model

OpenGL lets you manipulate two matrices: the modelview matrix and the
projection matrix. 

* The modelview matrix is the amalgamation of each object's world matrix with 
  the camera's view matrix. In other words, modelview = view $\times$ model.
  Hence the name. As such, the modelview transforms object space to eye space.

* The projection matrix defines the transformation from eye space to clip
  space.

To transform from object space to clip space, you can multiply a vertex by
$projection \times modelview = projection \times view \times model$. 
This is sometimes called the ModelViewProjection matrix.

### Matrix Stack

OpenGL provides a matrix 'stack' to hold intermediate transformations. It's not
actually a stack _per se_, but it does provide FIFO ordering. The last 
transformation you specify on the OpenGL stack is the first that gets applied 
to the vertex.

There are two matrix stacks: one for the projection matrix and one for the
modelview. The projection stack is typically boring, so we'll focus on the 
modelview stack.

Why a stack? Think about the order vertices are multiplied by matrices:

1. The original vertex is multiplied by the world matrix, moving it to world space
2. The new vertex is multiplied by the view matrix, moving it to eye space

Note that the view matrix changes once (it's constant throughout the scene),
but the world matrix changes often (changes on a per-object basis). This is
what makes the stack useful: we can push the view matrix once, then push/pop
the world matrix once per object. Our transformation algorithm becomes:

    Switch to the projection stack
    Clear the stack
    Push the projection matrix

    Switch to the modelview stack
    Clear the stack
    Push the view matrix

    For each object
        Push the object's model matrix
        Render the object
        Pop the object's model matrix

One significant pitfall of the matrix stack is ALL operations must be specified
in FIFO order. So if you want to specify an object's model matrix as

1. Rotate 90 degrees
2. Translate 3 units up

you would need to reverse the `gl` calls:

    glTranslate(...);
    glRotate(...);

### Vertex and Fragment Shaders

Shaders are programs you can run on the GPU to perform certain steps of the
[rendering pipeline](http://www.opengl.org/wiki/Rendering_Pipeline_Overview).

The vertex shader is executed on each input vertex. It is given a vertex in
object space and is responsible for transforming the vertex to clip space. 

The graphics hardware moves the vertex from clip space to screen space, and
then rasterizes to identify which pixels are inside the screen-space triangle.

The fragment shader is then executed on each rasterized pixel, to determine
what color it should have.

This is a simplified view of the rendering pipeline, but it will suffice for
our discussion.

## Lighting and Transformations

Determing light colors involves several vector operations. For example, the
diffuse intensity of a pixel depends on the dot product between the normal
vector $N$ and the vector to the light, $L$. 

As per our discussion earlier in this article, $N$ and $L$ would have to be in
the same vector space for their dot product to have any meaning. One natural
question, then, is which space should we transform them to?

Typically world space and eye space are good candidates, as they are
scene-global and are reasonably easy to visualize. Unfortunately, transforming
to either is made difficult by the way OpenGL combines the model and view
matrices into one modelview matrix.

### The OpenGL Way

Consider Phong's lighting model for a single point light:

$color = ambient + diffuse * (N \cdot L) + specular * (E \cdot R)^{specular\\\_power}$

We have four vector to compute:

* $N$, the normal vector
* $L$, the vector from the vertex to the light source
* $E$, the vector from the vertex to the eye point
* $R$, which is $L$ reflected about $N$

We'll represent all the vectors in eye space, as OpenGL makes this the easiest
space to transform to.

* To compute $N$, we'll multiply the object-space normal by the modelview
  matrix. In GLSL, this is easy:

        vec3 N = gl_NormalMatrix * gl_Normal;

* To compute $L$, we need to know the vertex position and the light position,
  both in eye space. The problem then comes down to vector subtraction:

        vec4 vertex = gl_ModelViewMatrix * gl_Vertex;
        vec4 light = gl_LightSource[0].position;
        vec3 L = normalize(light - vertex).xyz;

* To compute $E$:

        vec3 eyepos = vec3(0.0); // by definition of eye space
        vec3 E = eyepos - vertex; 
        
        // or just
        vec3 E = -vertex.xyz;

* To compute $R$:

        vec3 R = reflect(L, N);

An astute reader might wonder how `gl_LightSource[0]` gets set in your C
application. When you call `glLightfv(GL_POSITION, ...)`, you specify the
light's position in world space; yet the light pops out in the shader in eye
space. How can this be?

When you call `glLightfv(GL_POSITION, ...)` OpenGL actually multiplies the
position you specified by the current modelview matrix. The assumption is
that, when you specify your lights, the modelview matrix == the view matrix.
Thus multiplying moves the light from world space to eye space.

That's why it's very important your draw loop takes the following form:

    Switch to the modelview stack
    Clear the stack
    Push the view matrix

    Set up your lights (again, even if they're not moving!)

    For each object
        Push the model matrix
        Render the object
        Pop the model matrix

If you swap the order of these operations, you'll likely get incorrect
transformations and a nice headache.

## Wrap-Up

Much of the OpenGL API was deprecated in OpenGL 3.0 and 'removed' in OpenGL
3.1. This functionality, which we used above extensively, is still available on
desktop computers via what OpenGL calls the 'compatability profile.'

If you opt for the core profile instead (or if you're using OpenGL ES or WebGL,
neither of which supports a compatibility profile), you have much more freedom. 
You can specify your model, view and projection matrices to your shaders as
separate uniforms. This freedom comes at the cost of boilerplate: you have to 
do much more work to render just a single triangle. 

Eventually, no matter what platform you're writing shaders on, being able to
think about your vectors in terms of spaces and transformations will come in
handy. This article should hopefully have been enough to get you started!

