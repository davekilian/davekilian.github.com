---
layout: post
title: Shader Transformations
author: Dave
draft: true
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

