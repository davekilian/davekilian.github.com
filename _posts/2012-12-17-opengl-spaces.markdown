---
layout: post
title: How Shader Transformations Work
author: Dave
draft: true
---

Shader programming requires some care, since bugs can be difficult to find.
Incorrect use of transformations is one of the most common shader bugs.
Without some background, it's also one of the easiest mistakes to make. 

Incorrect transformations can cause all sorts of problems:

* Lighting could look totally wrong.
* Lighting may look correct, but only from one point of view.
* It may appear as if the light source is moving with the camera.

Oftentimes beginners have a hard time figuring out which transformations to
apply to each vector. This post is an overview of vector spaces and proper
use of transformations in GLSL. 

We'll assume you have some experience with 3D graphics (specifically, lighting
models and per-object transformations). We'll also assume you have a little
experience writing shader programs.

Shall we?

## Starting with the Basics

$5 + 2 = 7$. Or does it? Things get murkier when we add units to the mix.

* If every term uses the same units, then the equation holds:
  $5 m + 2 m = 7 m$

* But if the units don't match, the equation is meaningless:
  $5 m + 2 ft = 7 (...?)$

We can, however, _transform_ each term from one unit space into another. You
can check your transformations are correct using a technique called
[dimensional analysis](http://www.chem.tamu.edu/class/fyp/mathrev/mr-da.html):

$5 m + 2 ft * \dfrac{0.3048 m}{ft} = 5.6096 m$

## Vector Spaces

Like units, vector spaces give meaning to values. We used units to give
physical meaning to numbers; we'll use vector spaces to do something similar
with vectors.

Before, we defined units relative to each other using scale factors. There were
.3048 meters in a single foot; ergo there were 0.6096 meters in two feet. 

We can do the same thing with vector spaces relative to each other. We can also
go further than just scaling. A few common ways of representing vector spaces 
relative to each other include:

* **Scaling**, as we just discussed.
* **Rotation**, where the direction of a vector in one space is the rotation of
  the same vector in a different space.
* **Translation**, where the tip of a vector (or a point in the vector space)
  is displaced a fixed amount along each of the axes relative to the same
  vector in a different space.
* Any combination of the above.

We can represent these transformations using matrices:

1. Given a vector $v$ in vector space $A$
2. And a matrix $M$ representing a transform from vector space $A$ to vector
   space $B$
3. The value of $v$ in vector space $B$ is $M \times v$ (or just $Mv$)

We mentioned we could combine transformations. This works as follows:

* Say we have a vector $v$, which is defined in space $A$
* We have a matrix $M\_1$ that transforms from $A$ to space $B$
* We have a matrix $M\_2$ that transforms from $B$ to space $C$
* We would like to computer $v$ in space $C$

To compute $v$ in $B$-space, we can multiply $v$ by $M\_1$. To compute $v$ in
$C$-space, we can multiply the result by $M\_2$. The final expression becomes
$M\_2(M\_1v)$, or $M\_2M\_1v$.

One handy property of matrices is associativity. This allows us to combine
multiple transformation matrices into a single transformation matrix. For
example, if we computed $M\_3 = M\_2M\_1$, then we can transform $v$ from
$A$-space to $C$-space with just one multiplication: $M\_3v$

Another handy property is invertibility. If $M\_1$ transforms from $A$-space to
$B$-space, the inverse of $M\_1$ (denoted $(M\_1)^{-1}$) transforms from
$B$-space to $A$-space.

### Why is Any of This Important?

Before, we noted that it makes no sense to directly add two numbers that have
different units. Similarly, it makes no sense to use two vectors in the same
equation (addition, subtraction, dot products, what have you) unless both
vectors are defined in the same vector space. 

## Common Vector Spaces in Graphics

In real-time rendering (using packages like OpenGL and DirectX), each primitive
goes through a series of vector spaces to end up on your screen. Here are the
important ones:

### Object Space

Most scenes are composed either from primitives (e.g. spheres and cylinders), 
models (i.e. triangle meshes) or both. An "object" is simply one of your 
primitives or one of your models. 

Object space is a coordinate system that is convenient for defining a specific
object in your scene. Typically, object spaces are defined such that the object
is at the origin. 

Unlike the rest of the spaces listed below, there are multiple object spaces.
There is one object space for each unique primitive / model in your scene.

### World Space

To compose a scene from objects, the objects need to be defined relatively to
one another. As such, they need to each be transformed into one common vector
space. We call this space world space.

World space is typically defined so the center of the scene is at the origin.
Sometimes the units in world space correspond to physical length.

### Eye Space

(also known as camera space, view space)

The camera represents the position and orientation from which the viewer is
observing the scene. Eye space is simply the object space for the camera.

It turns out representing the scene in eye space makes it easier to render. As
such, we typically end up transforming everything in the scene from world space
to eye space.

### Clip Space

Clip space is defined as what the viewer sees, or more specifically, exactly
what we will be rendered to the display. 

* The origin of clip space is defined as the center of your monitor. 
* $x = -1$ corresponds to the left boundary of your monitor.
* $x = 1$ corresponds to the right boundary
* $y = -1$ corresponds to the bottom
* $y = 1$ corresponds to the top

The $z$ axis in clip space is used for depth buffering. $z=0$ correspods to the
near plane, and $z = -1$ corresponds to the far plane.

### Screen Space

Screen space is the only 2D space in this list. Screen space is defined such
that

* $(0, 0)$ corresponds to the top-left corner of your display
* $(w-1, 0)$ corresponds to the top-right
* $(0, h-1)$ corresponds to the bottom-left
* $(w-1, h-1)$ corresponds to the bottom-right

Where $w$ and $h$ are respecitvely the width and height of your display in 
pixels.

## Transforming Between the Spaces

Recall that transformations between vector spaces are represented with
matrices. The goal of rendering is to transform each triangle from its original
object-space vertex positions to screen-space pixel positions.

The following transformation pipeline accomplishes this.

### Object $\rightarrow$ World

Called the *world* or *model* matrix. A composition of transformations
(typically translations, rotations and scales) that defines the location of
each object in the scene.

Since there is one object space for each object, there is also one world matrix
for each object. World matrices can be one-off computations of the output of a
specialized data structure like a 
[scene graph](http://en.wikipedia.org/wiki/Scene_graph).

### World $\rightarrow$ Eye

Called the *view* matrix. A composition of (typically) translations and
rotations that move the scene so that the camera lies in eye space.

Since the camera has its own eye space and has a position/orientation in the
scene, you can construct a world matrix for the camera. This matrix would
transform from the camera's object space (a.k.a. eye space) to world space.

The view matrix is the inverse of such a transformation. 

### Eye $\rightarrow$ Clip

Called the *projection* matrix. The projection matrix is usually the
composition of a scaling operation followed by a perspectivization:

* The scale operation scales down the scene so the visible range spans from -1
  to 1 on the $x$ and $y$ axes, as defined by clip space.
* The perspectivization operation "unhinges" the scene, making closer objects
  seem bigger. This simulates visual perspective.

### Clip $\rightarrow$ Screen

Usually handled by the graphics package, not exposed to the programmer. Simply
drops the $Z$ term and scales the $X$ and $Y$ terms appropriately.

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

Note that we want to apply the model matrix to each vertex before the view
matrix. This will successfully transform the vertex from object space to eye
space. In stack semantics this means:

1. Pushing the view matrix onto the stack
2. Then pushing the object's model matrix onto the stack

Now we see why stack semantics are useful: we can specify the view matrix just
once. Our transformations algorithm becomes:

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

The graphics hardware moves the vertex to screen space and computes which
pixels are inside the screen-space triangle. It executes the fragment shader on
each pixel to determine what color it should have.

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

$color = ambient + diffuse * (N \cdot L) + specular * (E \cdot R)^{specular\\_power}$

We have four vector to compute:

* $N$, the normal vector
* $L$, the vector from the vertex to the light source
* $E$, the vector from the vertex to the eye point
* $R$, $L$ reflected about $N$

We'll represent all the vectors in eye space, as OpenGL makes this the easiest
space to transform to.

* To compute $N$, we'll multiply the object-space normal by the modelview
  matrix. In GLSL, this is easy:

        N = gl_NormalMatrix * gl_Normal;

* To compute $L$, we need to know the eye-space vertex position and the eye
  space light position. The problem then comes down to vector subtraction:

        vertex = gl_ModelViewMatrix * gl_Vertex;
        light = gl_LightSource\\[0\\].position
        L = normalize(light - vertex)

* To compute $E$:

        eye = vec3(0.0); // by definition of eye space
        E = eye - vertex; 
        
        // or just
        E = -vertex

* To compute $R$:

        R = reflect(L, N)

An astute reader might wonder how `gl_LightSource[0]` gets set in your C
application. When you call `glLightfv(GL_POSITION, ...)`, you specify the
light's position in world space; yet the light pops out in the shader in eye
space. How can this be?

When you call `glLightfv(GL_POSITION, ...)` OpenGL actually multiplies the
position you specified with the current modelview matrix. The assumption is
that, when you specify your lights, the modelview matrix == the view matrix.
Thus the modelview transforms from world to eye, and the resulting light
position is in eye space.

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
transformations and a moderately severe headache.

## Wrap-Up

Much of the OpenGL API was deprecated in OpenGL 3.0 and 'removed' in OpenGL
3.1. This functionality, which we used above exnteisvely, is still available on
the desktop via the Compatability Profile.

If you opt for the core profile instead (or if you're using OpenGL ES or WebGL,
neither of which supports a compatibility profile), you have much more freedom. 
You can specify your model, view and projection matrices to your shaders as
separate uniforms. 

This freedom comes at the cost of boilerplate: you have to do much more work to
render just a single triangle. You can find lots more using your favorite
search engine.

Eventually, no matter what platform you're writing shaders on, being able to
think about your vectors in terms of spaces and transformations will come in
handy. This article should hopefully have been enough to get you started!

