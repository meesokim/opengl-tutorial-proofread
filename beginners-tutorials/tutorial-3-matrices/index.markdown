---
layout: page
status: publish
published: true
title: 'Tutorial 3 : Matrices'
date: '2011-04-09 20:59:15 +0200'
date_gmt: '2011-04-09 20:59:15 +0200'
categories: [tuto]
order: 30
tags: []
---
{:TOC}

> _The engines don't move the ship at all. The ship stays where it is and the engines move the universe around it._
> 
> Futurama

**This is the single most important tutorial of the whole set. Be sure to read it at least eight times.**

# Homogeneous coordinates

Until now, we only considered 3D vertices as a (x,y,z) triplet, and tucked a constant 1.0 at the end in the vertex shader for reasons we didn't make clear. Let's introduce w. We will now have (x,y,z,w) vectors for all vertex coordinates.

This will be more clear soon, but for now, just remember this :

- If w == 1, then the vector (x,y,z,1) is a _position_ in space.
- If w == 0, then the vector (x,y,z,0) is a _direction_.

(In fact, remember this forever.)

What difference does this make? Well, for a rotation, it doesn't change anything. When you rotate a point or a direction, you get the same result. However, for a translation (when you move the point in a certain direction), things are different. What would it mean to "translate a direction"? Not much. Directions should not be translated at all.

Homogeneous coordinates allow us to use a single mathematical formula to deal with these two cases and, perhaps most importantly, to express translations as a matrix multiplication along with rotation and scaling.

# Transformation matrices

## An introduction to matrices

Simply put, a matrix is an array of numbers with a predefined number of rows and colums. For instance, a 2x3 matrix can look like this :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/2X3.png)

In 3D graphics we will mostly use 4x4 matrices. They will allow us to transform our (x,y,z,w) vertices. This is done by multiplying the vertex with the matrix :

**Matrix x Vertex (in this order!!) = TransformedVertex**

![]({{site.baseurl}}/assets/images/tuto-3-matrix/MatrixXVect.gif)

This isn't as scary as it looks. Put your left finger on the a, and your right finger on the x. This is _ax_. Move your left finger to the next number (b), and your right finger to the next number (y). You've got _by_. Once again : _cz_. Once again : _dw_. ax + by + cz + dw. You've got your new x ! Do the same for each line, and you'll get your new (x,y,z,w) vector.

Now this is sixteen multiplications, which is quite boring to compute, an we will do this often, so let's ask the computer to do it instead.

**In C++, with GLM:**

``` cpp
glm::mat4 myMatrix;
glm::vec4 myVector;
// fill myMatrix and myVector somehow
glm::vec4 transformedVector = myMatrix * myVector; // Again, in this order! This is important.
```

**In GLSL :**

``` glsl
mat4 myMatrix;
vec4 myVector;
// fill myMatrix and myVector somehow
vec4 transformedVector = myMatrix * myVector; // Yeah, it's pretty much the same as GLM
```

(Have you cut'n pasted this in your code? Go on, try it!)

## Translation matrices

These are the most simple tranformation matrices to understand. A translation matrix look like this :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/translationMatrix.png)

where X,Y,Z are the values that you want to add to your position.

So if we want to translate the vector (10,10,10,1) of 10 units in the X direction, we get :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/translationExamplePosition1.png)

(Do it! Doooooo it)

... and we get a (20,10,10,1) homogeneous vector! Remember, the 1 means that it is a position, not a direction. So our transformation didn't change the fact that we were dealing with a position, which is good.

Let's now see what happens to a vector that represents a direction towards the -z axis : (0,0,-1,0)

![]({{site.baseurl}}/assets/images/tuto-3-matrix/translationExampleDirection1.png)

... i.e. our original (0,0,-1,0) direction, which is great because as I said ealier, moving a direction does not make sense.

So, how does this translate to code?

**In C++, with GLM:**

``` cpp
#include <glm/gtx/transform.hpp> // after <glm/glm.hpp>
 
glm::mat4 myMatrix = glm::translate(10.0f, 0.0f, 0.0f);
glm::vec4 myVector(10.0f, 10.0f, 10.0f, 0.0f);
glm::vec4 transformedVector = myMatrix * myVector; // guess the result
```

**In GLSL :**

``` glsl
vec4 transformedVector = myMatrix * myVector;
```

Why don't we create the matrix in GLSL? Well, we could do it by setting each of the 16 elements in the matrix to the correct value, but in fact, you almost never create matrices in GLSL. Most of the time, you use some utility function like glm::translate() in C++ to compute your matrix, send it to GLSL, and do only the multiplication in the shader code.

## The Identity matrix

This one is special. It doesn't do anything. But I mention it because it's as important as knowing that multiplying A by 1.0 gives A.

![]({{site.baseurl}}/assets/images/tuto-3-matrix/identityExample.png)

**In C++ :**

``` cpp
glm::mat4 myIdentityMatrix = glm::mat4(1.0f);
```

## Scaling matrices

Scaling matrices are quite easy too :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/scalingMatrix.png)

So if you want to scale a vector (position or direction, it doesn't matter) by 2.0 in all directions:

![]({{site.baseurl}}/assets/images/tuto-3-matrix/scalingExample.png)

and the w still didn't change. You may ask : what is the meaning of "scaling a direction" ? Well, often, not much, so you usually don't do such a thing, but in some (rare) cases it can be handy.

(Notice that the identity matrix can be seen as a special case of scaling matrices, with (X,Y,Z) = (1,1,1). It's also a special case of a translation matrix with (X,Y,Z)=(0,0,0), by the way.)

**In C++ :**

``` cpp
// Use #include <glm/gtc/matrix_transform.hpp> and #include <glm/gtx/transform.hpp>
glm::mat4 myScalingMatrix = glm::scale(2.0f, 2.0f ,2.0f);
```

## Rotation matrices

These are quite complicated. I'll skip the details here, as it's not important to know their exact layout for everyday use. For more information, please have a look to the [Matrices and Quaternions FAQ]({{site.baseurl}}/assets/faq_quaternions/index.html) (popular resource, probably available in your language as well). You can also have a look at the [Rotations tutorials]({{site.baseurl }}{{intermediate-tutorials/tutorial-17-quaternions}}) 

**In C++ :**

``` cpp
// Use #include <glm/gtc/matrix_transform.hpp> and #include <glm/gtx/transform.hpp>
glm::vec3 myRotationAxis( ??, ??, ?? );
glm::rotate( angle_in_radians, myRotationAxis );
```

## Cumulating transformations

So now we know how to rotate, translate, and scale our vectors. It would be great to combine these transformations. This is done by multiplying the matrices together, for instance :

``` cpp
TransformedVector = TranslationMatrix * RotationMatrix * ScaleMatrix * OriginalVector;
```

**!!! BEWARE !!!** This lines actually performs the scaling FIRST, and THEN the rotation, and THEN the translation. This is how matrix multiplication works. The matrix that is closest to the vector (to the right) affects it first.

Writing the operations in another order wouldn't produce the same result. Try it yourself :

- take one step ahead ( beware of your computer ) and turn left;

- turn left, and take one step ahead

As a matter of fact, the order above is what you will usually need for game characters and other items : Scale it first if needed; then set its direction, then translate it. For instance, given a ship model (rotations have been removed for simplification) :

* The wrong way :
	- You translate the ship by (10,0,0). Its center is now at 10 units of the origin.
	- You scale your ship by 2\. Every coordinate is multiplied by 2 _relative to the origin_, which is far away. So you end up with a big ship, but centered at 2*10 = 20. Which you don't want.

* The right way :
	- You scale your ship by 2\. You get a big ship, centered on the origin.
	- You translate your ship. It's still the same size, and at the right distance.

Matrix-matrix multiplication is very similar to matrix-vector multiplication, so I'll once again skip some details and redirect you the the [Matrices and Quaternions FAQ]({{site.baseurl}}/assets/faq_quaternions/index.html#Q11) if needed. For now, we'll simply ask the computer to do it:

**in C++, with GLM :**

``` cpp
glm::mat4 myModelMatrix = myTranslationMatrix * myRotationMatrix * myScaleMatrix;
glm::vec4 myTransformedVector = myModelMatrix * myOriginalVector;
```

**in GLSL :**

``` glsl
mat4 transform = mat2 * mat1;
vec4 out_vec = transform * in_vec;
```

# The Model, View and Projection matrices

_For the rest of this tutorial, we will suppose that we know how to draw Blender's favourite 3d model: the monkey Suzanne._ You will learn how to draw this kind of object in a later tutorial, but bear with us. A single triangle is a little too simple to show the transformations clearly enough.

A handy method to keep track of your OpenGL transformations is to separate your matrices cleanly into three parts: the Model, View and Projection matrices. There is nothing in OpenGL or GLSL that forces you to do this, and you may choose not to do it (after all, that's what we did in tutorials 1 and 2). But you should. This is the way everybody does it, because it's easier this way.

## The Model matrix

This model, just as our beloved red triangle, is defined by a set of vertices. The X,Y,Z coordinates of these vertices are defined relative to the object's center : that is, if a vertex is at (0,0,0), it is at the center of the object.

![]({{site.baseurl}}/assets/images/tuto-3-matrix/model.png)

We'd like to be able to move this model, maybe because the player controls it with the keyboard and the mouse. Easy, you just learnt do do so : `translation*rotation*scale`, and done. You apply this matrix to all your vertices at each frame (in GLSL, not in C++!) and everything moves. Something that doesn't move will be at the _center of the world_.

![]({{site.baseurl}}/assets/images/tuto-3-matrix/world.png)

Your vertices are now in _World Space_. This is the meaning of the black arrow in the image below : _We went from Model Space (all vertices defined relatively to the center of the model) to World Space (all vertices defined relatively to the center of the world)._

![]({{site.baseurl}}/assets/images/tuto-3-matrix/model_to_world.png)

We can sum this up with the following diagram :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/M.png)

## The View matrix

Let's quote Futurama again:

> _The engines don't move the ship at all. The ship stays where it is and the engines move the universe around it._

![]({{site.baseurl}}/assets/images/tuto-3-matrix/camera.png)

When you think about it, the same applies to cameras. It you want to view a mountain from another angle, you can either move the camera... or move the mountain. While not practical in real life, this is really simple and handy in Computer Graphics. The OpenGL "camera" always stays put at the origin, looking at the (x,y) plane along the negative Z axis. It is your job to _transform the world_ to make it appear in the desired position, orientation and size in front of the camera.

So initially your camera is at the origin of the World Space. In order to move the world, you simply introduce another matrix. Let's say you want to move your camera of 3 units to the right (+X). This is equivalent to moving your whole world (and all objects in it) 3 units to the _left_! (-X). While your brain melts, let's just do it:

``` cpp
// Use #include <glm/gtc/matrix_transform.hpp> and #include <glm/gtx/transform.hpp>
glm::mat4 ViewMatrix = glm::translate(-3.0f, 0.0f ,0.0f);
```

Again, the image below illustrates this: _We went from World Space (all vertices defined relative to the center of the world, as we did in the previous section) to Camera Space (all vertices defined relative to the camera)._

![]({{site.baseurl}}/assets/images/tuto-3-matrix/model_to_world_to_camera.png)

Before you head explodes from this, enjoy GLM's great glm::lookAt function, which makes it a lot easier to define a view matrix in a way human beings are better suited to understanding:

``` cpp
glm::mat4 CameraMatrix = glm::lookAt(
    cameraPosition, // The position of your camera, in world space
    cameraTarget,   // Where you want to look at, in world space
    upVector        // Usually glm::vec3(0,1,0), but (0,-1,0) would make you look upside-down, which can be great too
);
```

Here's the compulsory diagram :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/MV.png)

This is not over yet, though.

## The Projection matrix

We're now in Camera Space. This means that after all these transformations, a vertex that happens to have x==0 and y==0 should be rendered at the center of the screen. But we can't use only the x and y coordinates to determine where an object should be put on the screen: its distance to the camera (z) counts, too! For two vertices with similar x and y coordinates, the vertex with the biggest negative z coordinate (farthest from the camera) will be more to the center of the screen than the other.

This is called a _perspective projection_:

![]({{site.baseurl}}/assets/images/tuto-3-matrix/model_to_world_to_camera_to_homogeneous.png)

Luckily for us, a 4x4 matrix can be used (with some tricks) to represent this projection:

``` cpp
// Generates a really hard-to-read matrix, but a standard 4x4 matrix nonetheless
glm::mat4 projectionMatrix = glm::perspective(
    FoV,         // The horizontal Field of View, in radians : the amount of "zoom". Think "camera lens". Usually between pi/2 (90°, extra wide) and pi/6 (30°, quite zoomed in)
    4.0f / 3.0f, // Aspect Ratio, the width-to-height proportions of your window. 4/3 == 800/600 == 1280/960, sounds familiar?
    0.1f,        // Near clipping plane. Must be > 0. Keep as big as possible, or you'll get precision issues.
    100.0f       // Far clipping plane. Keep as small as possible.
);
```

The trick is that after a coordinate vector is multiplied by a perspective projection matrix, the last coordinate (w) will not be 1.0, but the distance to the camera. In a separate step, which is performed automatically by OpenGL after the vertex shader, the (x,y,z) are _divided by w_. After this perspective division, which is performed without you writing any shader code for it, all vertices in the field of view and at distances between the near and far planes (points within the _view frustum_) are transformed to lie inside a cube with x, y and z all in the range from -1 to 1.

One last time :

_We went from Camera Space (all vertices defined relatively to the camera) to Homogeneous Space (all vertices defined in a small cube. Everything inside the cube is onscreen)._

And the final diagram :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/MVP.png)

Here's another diagram so that you understand better what happens with this Projection stuff. Before projection, we've got our blue objects, in Camera Space, and the red shape represents the frustum of the camera : the part of the scene that the camera is actually able to see.

![]({{site.baseurl}}/assets/images/tuto-3-matrix/nondeforme.png)

Multiplying everything by the Projection Matrix has the following effect:

![]({{site.baseurl}}/assets/images/tuto-3-matrix/homogeneous.png)

In this image, the frustum is now a perfect cube (between -1 and 1 on all axes, it's a little bit hard to see it), and all blue objects have been deformed in the same way. Thus, the objects that are near the camera (near the face of the cube that we can't see) are big, the others are smaller. Seems like real life!

Let's see what it looks like from "behind" the frustum:

![]({{site.baseurl}}/assets/images/tuto-3-matrix/projected1.png)

Here you get your image! It's just a little bit too square, so another mathematical transformation is applied (this one is automatic, you don't have to do it yourself in the shader) to fit this square to the actual window rectangle :

![]({{site.baseurl}}/assets/images/tuto-3-matrix/final1.png)

And *this* is, finally, the image that is actually rendered!

## Cumulating transformations : the ModelViewProjection matrix

... Just a standard matrix multiplication as you already love them!

``` cpp
// C++ : compute the matrix
glm::mat4 MVPmatrix = projection * view * model; // Remember : reverse order!
```

``` glsl
// GLSL : apply it
transformed_vertex = MVP * in_vertex;
```
{: .highlightglslfs }

# Putting it all together

* First step : generating our MVP matrix. This must be done for each model you render. The view and projection matrices usually stay the same for the entire scene, but the model matrix is different for each obejct.

  ``` cpp
  // Projection matrix : 45° Field of View, 4:3 ratio, display range : 0.1 unit <-> 100 units
  glm::mat4 Projection = glm::perspective(glm::radians(45.0f), (float) width / (float)height, 0.1f, 100.0f);
  
  // Or, for an ortho camera :
  //glm::mat4 Projection = glm::ortho(-10.0f,10.0f,-10.0f,10.0f,0.0f,100.0f); // In world coordinates
  
  // Camera matrix
  glm::mat4 View = glm::lookAt(
      glm::vec3(4,3,3), // Camera is at (4,3,3), in World Space
      glm::vec3(0,0,0), // and looks at the origin
      glm::vec3(0,1,0)  // Head is up (set to 0,-1,0 to look upside-down)
      );
  
  // Model matrix : an identity matrix (model will be at the origin)
  glm::mat4 Model = glm::mat4(1.0f);
  // Our ModelViewProjection : multiplication of our 3 matrices
  glm::mat4 mvp = Projection * View * Model; // Remember, matrix multiplication works "backwards" from the right
  ```

* Second step : give the matrix to GLSL

  ``` cpp
  // Get a handle for our "MVP" uniform
  // Only required once, after successful compilation of the shader
  GLuint MatrixID = glGetUniformLocation(program_id, "MVP");
  
  // Send our transformation to the currently bound shader, in the "MVP" uniform
  // This is done in the main loop since each model will have a different MVP matrix (At least for the M part)
  glUniformMatrix4fv(mvp_handle, 1, GL_FALSE, &mvp[0][0]);
  ```

* Third step : use the matrix in GLSL to transform our vertices

  ``` glsl
  // Input vertex data, different for all executions of this shader.
  layout(location = 0) in vec3 vertexPosition_modelspace;
  
  // Values that stay constant for the whole mesh.
  uniform mat4 MVP;
  
  void main(){
    // Output position of the vertex, in clip space : MVP * position
    gl_Position =  MVP * vec4(vertexPosition_modelspace,1);
  }
  ```
  {: .highlightglslvs }

* Done! Here is the same triangle as in tutorial 2, still at the origin (0,0,0), but viewed in perspective from point (4,3,3), heads up (0,1,0), with a 45° field of view.

![]({{site.baseurl}}/assets/images/tuto-3-matrix/perspective_red_triangle.png)

In tutorial 6 you'll learn how to modify these values dynamically using the keyboard and the mouse to create a game-like camera, but first, we'll learn how to give our 3D models some colour (tutorial 4) and textures (tutorial 5).

# Exercises

*   Try changing the glm::perspective
*   Instead of using a perspective projection, use an orthographic projection (glm::ortho)
*   Modify ModelMatrix to translate, rotate, then scale the triangle
*   Do the same thing, but in different orders. What do you notice? What is the "best" order that you would want to use for a character?

_Footnotes_

[^projection]:A perspective transformation is not affine, and as such, can't be represented entirely by a matrix. After being multiplied by the ProjectionMatrix, homogeneous coordinates are divided by their own W component. This W component happens to be -Z (because the projection matrix has been crafted this way). This way, points that are far away from the origin are divided by a big Z; their X and Y coordinates become smaller; points become more close to each other, objects seem smaller; and this is what gives the perspective. This perspective division is done in hardware, and is not visible in the shader.
