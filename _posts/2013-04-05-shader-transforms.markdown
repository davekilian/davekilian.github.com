---
layout: post
title: OpenGL Shader Transformations
author: Dave
draft: true
---

OpenGL is a complex system. In my 8 semesters of undergrad study, I spent about
6 of them working with OpenGL in some respect. Only about halfway through that 
did I finally start to feel like things were starting to make sense!

One of the bigger stumbling blocks I ran into was figuring out how to properly
use transformations with GLSL shaders. It turns out there's just about one
'correct' way to set up a shader-based OpenGL app, and plenty of ways to mess
up. The result of messing up is usually something plausible-looking yet 
bizarrely broken -- like a scene that seems fine until you realize moving the 
camera moves the lights with it.

These mistakes are all too common, and I've never found a single resource that
covers the issue end to end. In this post, I'll try to rectify that! We'll
be talking about transformations quite a bit, so if you're a bit rusty, or if
the terms don't seem familiar, I'd recommend reading about the
[Transformation Pipeline](http://www.davekilian.com/2013/04/27/transformation-pipeline.html).

## A History Lesson (Optional)

OpenGL's shader story is, admittedly, a little messed up.

OpenGL was born in the heydey of fixed-function hardware. Fixed-function
graphics cards, unlike modern cards, had 'hardcoded' circuitry for performing
just about any graphics-related task. The graphics programmer simply toggled
specific features on or off, and then sent some vertices to the card to be
rendered.

Nowadays pretty much all graphics cards are shader-based. These cards don't
know a ton about rendering; instead, they just execute shader programs.
These shaders contain the actual rendering logic. 

Back in the fixed-function days, only graphics card manufacturers (like ATI and
NVIDIA) were able to add new rendering features to their cards. Developers had 
to wait for new cards to support new features, and users had to upgrade to the 
newest cards to use the newest features. With the advent of shaders, it's now 
possible for developers to invent new rendering features and run them on 
existing hardware. 

OpenGL was originally invented to be a common bridge between different
fixed-function systems. As long as each graphics manufacturer implented OpenGL
for its fixed-function hardware, any OpenGL-based app could run on any
OpenGL-compatible hardware, even if the individual cards worked vastly
differently. 

As the tides turned from fixed-function to shader-based graphics, OpenGL
needed to evolve. OpenGL 2.0 added support for a GL shading language (GLSL).
While this opened up a wide range of possibilities to graphics developers, the
choice to keep shaders alongside the older functionality limited shaders'
flexibility. OpenGL 3.0 deprecated most of this older
functionality, and OpenGL 3.1 removed it. Nowadays, OpenGL 4.x (the latest 
version at the time of writing) is more or less a thin wrapper for loading 
shaders and graphics data (basically, vertices and textures) onto the graphics 
card and executing the shaders. However, in some cases developers have access
to the OpenGL Compatibility Profile, which implements the older functionality
removed in GL 3.x.

The Khronos Group (which develops and maintains OpenGL) only intended for the
compatibility profile to allow older OpenGL apps to continue working; however,
a lot of courses and tutorials still use the compatibility profile for
teaching, since it requires less boilerplate to write an OpenGL 'Hello World'
app. Unfortunately, the compatibility profile retains the confusing and
poorly-documented interaction between the fixed-function API and shaders, which
confuses students when said tutorials move on to GLSL.

The goal of this post, then, is to explain how the fixed-function API interacts
with the shader system in the OpenGL compatibility profile. We'll develop a
correctly-formed shader-based 'Hello World' and explain why it works, and how
small changes cause completely diffrent behavior.

## TODO

* Talk about how vertex shaders play into the pipeline, what their role is,
  what code they replace, `ftransform()`

* Do everything in eye space

* The matrix stack is a stack -- remember to perform your operations in reverse
  order

* The modelview / projection split (the projection matrix is not the camera
  matrix!)

* Getting eye-space coordinates in shaders

* A minimal working example of blinn-phong or something

---

# Old Stuff

The stuff below was trimmed off an older draft of the transformations post

---

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

