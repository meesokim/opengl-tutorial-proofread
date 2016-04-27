---
layout: page
status: publish
published: true
title: 'Tutorial 6 : Keyboard and Mouse'
date: '2011-05-08 08:26:13 +0200'
date_gmt: '2011-05-08 08:26:13 +0200'
categories: [tuto]
order: 60
tags: []
---

Welcome for our 6th tutorial !

We will now learn how to use the mouse and the keyboard to move the camera just like in a FPS ("first person shooter").

# The interface

Since this code will be re-used throughout the tutorials, we will put the code in a separate file : common/controls.cpp, and declare the functions in common/controls.hpp so that tutorial06.cpp knows about them.

The code of tutorial06.cpp doesn't change much from the previous tutorial. The major modification is that instead of computing the MVP matrix once, we now have to do it every frame. So let's move this code inside the main loop:

``` cpp
do{

    // ...

    // Compute the MVP matrix from keyboard and mouse input
    computeMatricesFromInputs();
    glm::mat4 ProjectionMatrix = getProjectionMatrix();
    glm::mat4 ViewMatrix = getViewMatrix();
    glm::mat4 ModelMatrix = glm::mat4(1.0);
    glm::mat4 MVP = ProjectionMatrix * ViewMatrix * ModelMatrix;

    // ...
}
```

This code needs 3 new functions :

* computeMatricesFromInputs() reads the keyboard and mouse and computes the Projection and View matrices. This is where all the magic happens.
* getProjectionMatrix() returns the computed Projection matrix.
* getViewMatrix() returns the computed View matrix.

This is just one way to do it, of course. If you don't like these functions, go ahead and change them. Just make sure you keep the projection matrix P and the view matrix V separate, and see to it that P only does the perspective projection and nothing else. This is because we often need to know the camera position and orientation in world space, and V describes this in the manner we expect, as a combination of rotation and translation. P performs a trick to make objects appear smaller at a distance, but the perspective transformation is highly non-linear to the point where angles and distances in camera space no longer make much sense.

Let's see what's inside controls.cpp.

# The actual code

We'll need a few variables.

``` cpp
// position
glm::vec3 position = glm::vec3( 0, 0, 5 );
// horizontal angle : toward -Z
float horizontalAngle = 3.14f;
// vertical angle : 0, look at the horizon
float verticalAngle = 0.0f;
// Initial Field of View
float initialFoV = 45.0f;

float speed = 3.0f; // 3 units / second
float mouseSpeed = 0.005f;
```

FoV is the level of zoom. 80&deg; = very wide angle, huge deformations. 60&deg; - 45&deg; : standard. 20&deg; : big zoom. Note that recent versions of glm measure angles in radians, not degrees.

We will first recompute _position_, _horizontalAngle_, _verticalAngle_ and _FoV_ according to the inputs, and then compute the View and Projection matrices from those values.

## Orientation

Reading the mouse position is easy :

``` cpp
// Get mouse position
int xpos, ypos;
glfwGetMousePos(&xpos, &ypos);
```

Hwever, for a first person shooter, you usually don't want the mouse pointer to be visible, and you don't want it to slip outside of the window when you look around. You can solve this by putting the cursor back to the center of the screen with the function glfwSetMousePos(), but a better method is to set the cursor mode to "hidden":

``` cpp
// Lock the mouse to this window, and hide the cursor
glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
```

The coordinates reported by getMousePos() are then no longer restricted to the window. They can be any number, and you can move the mouse any distance without losing focus for your OpenGL window.

We can make our code compute the angles from the absolute cursor position, but a more robust way is to remember where the mouse was in the previous frame, compute its relative movement and update the rotation angles accordingly. However, we can also reset the mouse position to (0,0) after each readout and just see how far it moved next time we read it. It's a bit ugly, but it saves us the hassle of declaring our own persistent variables for the previous position. With this approach, we must also remember to call glfwSetMousePosition(0,0) right before we enter our rendering loop for the first time, or the angles will make a big jump at the first frame. You might want to take a different approach, but this works OK for our purposes.

``` cpp
glfwGetMousePos(&xpos, &ypos);
glfwSetMousePos(0,0);
// Compute new orientation
horizontalAngle += mouseSpeed * (float)xpos;
verticalAngle   += mouseSpeed * (float)ypos;
```

Let's read this from right to left :

* xpos means : how far is the mouse from the (0,0) position? The bigger this value, the more we want to turn.
* float(...) converts it to a floating-point number so that the multiplication goes well.
* mouseSpeed is just there to set the mouse sensitivity to speed up or slow down the rotations. Fine-tune this at will, let the user choose it, or make it proportional to the window size for a more consistent user experience across different resolutions.
* += : If you didn't move the mouse, xpos will be 0, and horizontalAngle+=0 doesn't change horizontalAngle. If you had a "=" instead, you would be forced back to your original orientation each frame, which isn't what we want.

We can now compute a vector that represents, in World Space, the direction in which we're looking:

``` cpp
// Direction : Spherical coordinates to Cartesian coordinates conversion
glm::vec3 forward(
    cos(verticalAngle) * sin(horizontalAngle),
    sin(verticalAngle),
    cos(verticalAngle) * cos(horizontalAngle)
);
```

This is a standard trigonometric computation, but if you don't know about cosine and sine, here's a short explanation :

<img class="alignnone whiteborder" title="Trigonometric circle" src="http://www.numericana.com/answer/trig.gif" alt="" width="150" height="150" />

The formula above is just a generalisation to 3D, _spherical polar angles_, which is a very common way of representing directions in space.

Now we want to compute the "up" vector reliably. Notice that "up" isn't always towards +Y : if you look down, for instance, the "up" direction relative t the camera will be in fact horizontal. Here is an example of two cameras with the same position, the same target, but a different "up":

![]({{site.baseurl}}/assets/images/tuto-6-mouse-keyboard/CameraUp.png)

In our case, the only constant is that the vector that points to the right of the camera is always horizontal. You can check this by putting your arm horizontal, and looking up, down, in any direction. So let's define the "right" vector : its Y coordinate is 0 since it's horizontal, and its X and Z coordinates are just like in the figure above, but with the angles rotated by 90&deg;, or Pi/2 radians.

``` cpp
// Right vector
glm::vec3 right = glm::vec3(
    sin(horizontalAngle - 3.14f/2.0f),
    0,
    cos(horizontalAngle - 3.14f/2.0f)
);
```

We have a "right" vector and a "forward", or "view" vector. The "up" vector is a vector that is perpendicular to these two. A useful mathematical tool makes this very easy to compute: the _cross product_.

``` cpp
// Up vector : perpendicular to both forward and right
glm::vec3 up = glm::cross( right, forward );
```

To remember what the cross product does, it's very simple. Just recall the Right Hand Rule from Tutorial 3. The first vector is the thumb; the second is the index; and the result is the middle finger. It's very handy. Note that swapping the order of the two vectors in the cross product will flip the result, so make sure to use this right-hand rule correctly.

## Position

The code for moving the camera is pretty straightforward. For international readers, I used the up/down/right/left arrow keys instead of the "wasd" because on my "azerty" keyboard, "wasd" is actually "zqsd". It's also different with "qwerz" keyboards, let alone Cyrillic or Korean keyboards. (I don't even know what layout they have, but I guess it's very different.)

``` cpp
// Move forward
if (glfwGetKey( GLFW_KEY_UP ) == GLFW_PRESS){
    position += forward * deltaTime * speed;
}
// Move backward
if (glfwGetKey( GLFW_KEY_DOWN ) == GLFW_PRESS){
    position -= forward * deltaTime * speed;
}
// Strafe right
if (glfwGetKey( GLFW_KEY_RIGHT ) == GLFW_PRESS){
    position += right * deltaTime * speed;
}
// Strafe left
if (glfwGetKey( GLFW_KEY_LEFT ) == GLFW_PRESS){
    position -= right * deltaTime * speed;
}
```

The only special thing here is the _deltaTime_. You don't always want to move, say, 1 unit each frame, for a simple reason:

* If you have a fast computer, and you run at 60 fps, you'd move of 60*speed units in 1 second
* If you have a slow computer, and you run at 20 fps, you'd move of 20*speed units in 1 second

Since having a better computer is not an excuse for going faster, you have to scale the distance by the "time since the last frame", or "deltaTime".

* If you have a fast computer, and you run at 60 fps, you'd move of 1/60 * speed units in 1 frame, so 1*speed in 1 second.
* If you have a slow computer, and you run at 20 fps, you'd move of 1/20 * speed units in 1 second, so 1*speed in 1 second.

which is much better. The variable deltaTime is very simple to compute using the built-in timer in GLFW. The function _glfwGetTime_ returns the number of seconds since the program was started, as a _double_ :

``` cpp
double currentTime, lastTime;

currentTime = glfwGetTime();
float deltaTime = float(currentTime - lastTime);
lastTime = currentTime; // Remember for next frame
```

## Field Of View

For fun, we can also bind the wheel of the mouse to the Field Of View, so that we can have a cheap zoom :

``` cpp
float FoV = initialFoV - 0.1 * glfwGetMouseWheel(); // 0.1 radians is around 5 degrees
```

After this line of code, you might also want to make sure that FoV stays within reasonable limits. It must be strictly above 0 degrees and strictly below 180 degrees for the perspective matrix to be meaningful, but it makes sense to restrict it to a more narrow range, like from 10 to 120 degrees. A reasonable value for initialFoV is Pi/3, or 60 degrees.

## Computing the matrices

Computing the matrices is now straightforward. We use the exact same functions as before, but with our new parameters.

``` cpp
// Projection matrix : Desired field of View, 4:3 ratio, depth range : 0.1 unit <-> 100 units
ProjectionMatrix = glm::perspective(FoV, 4.0f / 3.0f, 0.1f, 100.0f);
// Camera matrix
ViewMatrix       = glm::lookAt(
    position,           // Camera is here
    position + forward, // and looks here : at the same position, plus "forward"
    up                  // Our computed "up" vector
);
```

# Results

![]({{site.baseurl}}/assets/images/tuto-6-mouse-keyboard/moveanim.gif)


## Backface Culling

Now that you can freely move around, you'll notice that if you go inside the cube, polygons are still displayed. This can seem obvious, but this remark actually opens an opportunity for optimisation. As a matter of fact, in a typical application, you are never _inside_ an object.

The idea is to let the GPU check if the camera is behind, or in front of, the triangle. If it's in front, display the triangle; if it's behind, *and* the mesh is closed, *and* we're not inside the mesh, *then* there will be another triangle in front of it, and nobody will notice anything, except that everything will be faster : half the amount of triangles to render!

The best thing is that it's very easy to check this. The GPU computes the _normal_ of the triangle, a vector that is perpendicular to its surface (using the cross product between two of its edges) and checks whether this normal is oriented towards the camera or not.

This comes at a cost, unfortunately: the orientation of the triangle is implicit and inferred from the _winding order_ of the triangle. What you need to do is make sure that the vertices for your triangles are specified in a counter-clockwise order as seen from the front of the triangle. This is the standard from every 3-D modeling program, but you need to consider it when you create triangle meshes in your own code. If you accidentally swap two vertices in your buffer, you'll end up with a hole because OpenGL thinks that the affected triangle is facing the other way. But backface culling is definitely worth your extra work.

Enabling backface culling is a breeze:

``` cpp
// Cull triangles that are not facing the camera
glEnable(GL_CULL_FACE);
```

# Exercices

* Restrict verticalAngle so that you can't go upside-down
* Create a camera that rotates around the object ( position = ObjectCenter + ( radius * cos(time), height, radius * sin(time) ) ); bind the radius/height/time to the keyboard/mouse, or whatever
* Have fun !
