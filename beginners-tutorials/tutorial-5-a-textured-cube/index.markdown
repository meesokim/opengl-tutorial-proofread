---
layout: page
status: publish
published: true
title: 'Tutorial 5 : A Textured Cube'
date: '2011-04-26 07:55:58 +0200'
date_gmt: '2011-04-26 07:55:58 +0200'
categories: [tuto]
order: 50
tags: []
---

In this tutorial, you will learn :

* What are UV coordinates
* How to load textures yourself
* How to use them in OpenGL
* What is filtering and mipmapping, and how to use them
* How to load texture more robustly with GLFW
* What the alpha channel is

# About UV coordinates

When texturing a mesh, you need a way to tell to OpenGL which part of the image has to be used for each triangle. This is done with UV coordinates.

Each vertex can have, in addition to its position, a couple of extra float coordinates, U and V. These coordinates are used to access the texture, in the following way:

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/UVintro.png)

Notice how the texture is distorted on the triangle. If you draw a mesh using its (u,v) coordinates for the (x,y) position on the screen, what you get is the so-called _UV map_ which describes how the texture image should be "pasted" or "painted" onto the 3-D model.

# Loading .BMP images yourself

There are several file formats for images, one of them being the old format BMP from Microsoft Windows. Knowing the BMP file format is not important: plenty of libraries can load BMP files for you. However, it's very simple to do it, and it can help you understand how things work under the hood. So we'll write a BMP file loader from scratch, so that you know how loading pixels from a file works, <span style="text-decoration: underline;">and then probably never use it again</span>.

Here is the declaration of the loading function :

``` cpp
GLuint loadBMP_custom(const char * imagepath);
```

and its intended use looks like this :

``` cpp
GLuint image = loadBMP_custom("./my_texture.bmp");
```

Let's see how to read a BMP file, then. Let's restrict ourselves to 3-component RGB files with 8 bits per component (24 bits per pixel) and no compression. (Our code will break miserably if we try to read any other kind of BMP file.) First, we'll need some variables to store information on the image size, and of course a place to store pixel data. These variables will be set when reading the file.

``` cpp
// Data read from the header of the BMP file
unsigned char header[54]; // Each BMP file begins by a 54-bytes header
unsigned int dataPos;     // Position in the file where the actual data begins
unsigned int width, height;
unsigned int imageSize;   // = width*height*3
// Actual RGB data
unsigned char * data;
```

We now have to actually open the file for reading. It's a binary file, not a text file, hence the "rb" mode.

``` cpp
// Open the file
FILE * file = fopen(imagepath,"rb");
if (!file){printf("Image could not be opened\n"); return 0;}
```

The first thing in a BMP file is a 54-byte header. It contains information such as "Is this file really a BMP file?", the size of the image, the number of bits per pixel, etc. So let's read this header :

``` cpp
if ( fread(header, 1, 54, file)!=54 ){ // If not 54 bytes read : problem
    printf("Not a correct BMP file\n");
    return false;
}
```

The header always begins with the two ASCII characters BM, which is a reasonably safe indication that this is indeed a BMP file. As a matter of fact, here's what you get when you open a .BMP file in a hexadecimal editor :

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/hexbmp.png)

So we have to check that the two first bytes are really 'B' and 'M' :

``` cpp
if ( header[0]!='B' || header[1]!='M' ){
    printf("Not a correct BMP file\n");
    return 0;
}
```

Now we can read the size of the image, the location of the data in the file, etc. These are encoded as 32-bit integers within the header, which we can extract with some old school C-style pointer manipulations:

``` cpp
// Read ints from the byte array
dataPos    = *(int*)&(header[0x0A]);
imageSize  = *(int*)&(header[0x22]);
width      = *(int*)&(header[0x12]);
height     = *(int*)&(header[0x16]);
```

BMP is a messy file format which is written wrong by some tools, so we might need to make up some info if it's missing :

``` cpp
// Some BMP files are misformatted, guess missing information
if (imageSize==0)    imageSize=width*height*3; // Assume one byte for each Red, Green and Blue component
if (dataPos==0)      dataPos=54; // Data starts past the BMP header
```

Now that we know the size of the image, we can allocate some memory to read the image into, and read the pixels:

``` cpp
// Create a buffer
data = new unsigned char [imageSize];

// Read the actual data from the file into the buffer
fread(data, 1, imageSize, file);

//Everything is in memory now, the file can be closed
fclose(file);
```

So much for reading pixels into CPU memory. We arrive now at the OpenGL part. Creating textures is very similar to creating vertex buffers: Create a texture, bind it, fill it with data, and configure it.

In glTexImage2D, the GL_RGB indicates that we are talking about a 3-component color, and GL_BGR says how exactly it is represented in RAM. For unclear reasons, BMP doesn't store its color components in the usual order (Red, Green, Blue) but instead (Blue, Green, Red), and we have to tell OpenGL about that, so that the data is reshuffled during upload to the GPU.

``` cpp
// Create one OpenGL texture
GLuint textureID;
glGenTextures(1, &textureID);

// "Bind" the newly created texture, All subsequent texture functions
// will modify this particular texture, until another texture is bound.
glBindTexture(GL_TEXTURE_2D, textureID);

// Send the image data to OpenGL (upload it to the GPU)
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_BGR, GL_UNSIGNED_BYTE, data);

// Configure the texture: tell OpenGL how it should be drawn in different resolutions
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
```

We'll explain those last two lines later. Meanwhile, on the C++-side, you can use your new function to load a texture:

``` cpp
GLuint Texture = loadBMP_custom("uvtemplate.bmp");
```

> Another important point: **use power-of-two dimensions for your textures!**
> 
> * good: 128 x 128, 256 x 256, 1024 x 1024, 2 x 2 ...
> * bad: 127 x 128, 3 x 5, 640 x 480...
> * also good: 128 x 256

Modern versions of OpenGL will handle textures of arbitrary size (up to some maximum), but there may be a speed penalty for using them, and non-power-of-two sizes are still not supported on all platforms.

# Using the texture in OpenGL

Once we have the data uploaded and the texture bound, using the texture in GLSL is simple. We'll have a look at the fragment shader first. Most of it is straightforward:

``` glsl
#version 330 core

// Interpolated texture coordinate values from the vertex shader
in vec2 UV;

// Ouput data
out vec3 color;

// Values that stay constant for the whole mesh.
uniform sampler2D myTextureSampler;

void main(){

    // Output color = color of the texture at the specified UV
    color = texture( myTextureSampler, UV ).rgb;
}
```

{: .highlightglslfs }

Three things :

* The fragment shader needs UV texture coordinates. Seems fair.
* It also needs a "sampler2D" in order to know which texture to access (you can access several textures in the same shader).
* Finally, accessing a texture is done with texture(), which gives back an (R,G,B,A) vec4. We'll see about the A shortly.

The vertex shader is simple too, you just have to pass the UVs from an additional vertex attribute to the fragment shader :

``` glsl
#version 330 core

// Input vertex data, different for all executions of this shader.
layout(location = 0) in vec3 vertexPosition_modelspace;
layout(location = 1) in vec2 vertexUV;

// Output data ; will be interpolated for each fragment.
out vec2 UV;

// Values that stay constant for the whole mesh.
uniform mat4 MVP;

void main(){

    // Output position of the vertex, in clip space : MVP * position
    gl_Position =  MVP * vec4(vertexPosition_modelspace, 1.0);

    // UV of the vertex. No special transformation for this one.
    UV = vertexUV;
}
```

{: .highlightglslvs }

Remember "layout(location = 0) in vec3  vertexPosition_modelspace" from Tutorial 4? Well, we'll have to do the exact same thing for the UV coordinates, but instead of filling a buffer with (R,G,B) triplets, we'll fill it with (U,V) pairs: "layout(location = 1) in vec2 vertexUV"

``` cpp
// Two UV coordinatesfor each vertex. They were created with Blender. You'll learn shortly how to do this yourself.
static const GLfloat g_uv_buffer_data[] = {
    0.000059f, 1.0f-0.000004f,
    0.000103f, 1.0f-0.336048f,
    0.335973f, 1.0f-0.335903f,
    1.000023f, 1.0f-0.000013f,
    0.667979f, 1.0f-0.335851f,
    0.999958f, 1.0f-0.336064f,
    0.667979f, 1.0f-0.335851f,
    0.336024f, 1.0f-0.671877f,
    0.667969f, 1.0f-0.671889f,
    1.000023f, 1.0f-0.000013f,
    0.668104f, 1.0f-0.000013f,
    0.667979f, 1.0f-0.335851f,
    0.000059f, 1.0f-0.000004f,
    0.335973f, 1.0f-0.335903f,
    0.336098f, 1.0f-0.000071f,
    0.667979f, 1.0f-0.335851f,
    0.335973f, 1.0f-0.335903f,
    0.336024f, 1.0f-0.671877f,
    1.000004f, 1.0f-0.671847f,
    0.999958f, 1.0f-0.336064f,
    0.667979f, 1.0f-0.335851f,
    0.668104f, 1.0f-0.000013f,
    0.335973f, 1.0f-0.335903f,
    0.667979f, 1.0f-0.335851f,
    0.335973f, 1.0f-0.335903f,
    0.668104f, 1.0f-0.000013f,
    0.336098f, 1.0f-0.000071f,
    0.000103f, 1.0f-0.336048f,
    0.000004f, 1.0f-0.671870f,
    0.336024f, 1.0f-0.671877f,
    0.000103f, 1.0f-0.336048f,
    0.336024f, 1.0f-0.671877f,
    0.335973f, 1.0f-0.335903f,
    0.667969f, 1.0f-0.671889f,
    1.000004f, 1.0f-0.671847f,
    0.667979f, 1.0f-0.335851f
};
```

The texture image is addressed by UV coordinates in the range 0.0 to 1.0, where (0.0, 0.0) is the lower left corner and (1.0, 1.0) is the upper right corner. The V coordinate is flipped in the code above to adjust for the fact that a BMP file, like most image file formats, stores pixels row by row from top to bottom and puts the origin at the top left instead of at the bottom left. The UV coordinates above correspond to the following model:

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/uv_mapping_blender.png)

The rest is the same as for the vertex coordinates: generate the buffer, bind it, fill it, configure it, and draw the Vertex Buffer as usual. Just be careful to use 2 as the second parameter (size) of glVertexAttribPointer instead of 3.

This is the result :

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/nearfiltering.png)

and a zoomed-in version :

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/nearfiltering_zoom.png)

# What is filtering and mipmapping, and how to use them

As you can see in the screenshot above, the texture quality is not that great. This is because in loadBMP_custom, we wrote :

``` cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
```

This means that in our fragment shader, texture() takes the value of the "texel" (texture pixel) that has its center closest to the specified (U,V) coordinates, and continues happily.

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/nearest.png)

There are several things we can do to improve this.

## Linear filtering

With linear filtering, texture() looks at four texels around the UV coordinate, and mixes the colours according to the distance to each of their centers. This avoids the hard pixelated edges seen above, but at the expense of some blur:

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/linear1.png)

This is much better, and this is used a lot, but if you want very high quality you can also use anisotropic filtering, which is a bit slower.

## Anisotropic filtering

This texture lookup method approximates the part of the texture image that is really seen through the fragment. For instance, if the following texture is seen from the side, and a little bit rotated, anisotropic filtering will compute the colour contained in the blue rectangle by taking a fixed number of samples (the "anisotropic level") along its main direction.

![]({{site.baseurl}}/assets/images/tuto-5-textured-cube/aniso.png)

## Mipmaps

Both linear and anisotropic filtering have a problem. If the texture is seen from far away, so mixing only 4 texels won't be enough. Actually, if your 3D model is so far away than it takes only 1 fragment on screen, ALL the texels of the image should be averaged to produce the final color. This is obviously not done at the time of texture lookup, for performance reasons. Instead, we introduce MipMaps:

![](http://upload.wikimedia.org/wikipedia/commons/5/5c/MipMap_Example_STS101.jpg)

* At texture initialisation time, you scale down your image by 2, successively, until you only have a 1x1 image (which effectively is the average of all the texels in the image)
* When you draw a mesh, you select which mipmap is the most appropriate, which is the size that has a texel size about the same size as a fragment. This selection is performed automatically by OpenGL.
* You sample this mipmap with either nearest, linear or anisotropic filtering.
* For additional quality, you can also sample the two mipmaps that match the fragment size most closely and blend the results. This is often called _trilinear filtering_, or in OpenGL jargon GL_LINEAR_MIPMAP_LINEAR.

Luckily, all this is very simple to do, OpenGL does everything for us provided that you ask it nicely:

``` cpp
// When MAGnifying the image (no bigger mipmap available), use LINEAR filtering
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
// When MINifying the image, use a LINEAR blend of two mipmaps, each filtered LINEARLY too
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
// Generate mipmaps, by the way. This is not performed unless you ask for it.
glGenerateMipmap(GL_TEXTURE_2D);
```

# How to load texture with GLFW

Our loadBMP_custom function is great because we made it ourselves, but using a dedicated library is better. GLFW version 2 could do that, but only for TGA files, and this feature has been removed in GLFW version 3, but here's how it could be done:

``` cpp
GLuint loadTGA_glfw(const char * imagepath){

    // Create one OpenGL texture
    GLuint textureID;
    glGenTextures(1, &textureID);

    // "Bind" the newly created texture : all future texture functions will modify this texture
    glBindTexture(GL_TEXTURE_2D, textureID);

    // Read the file, call glTexImage2D with the right parameters
    glfwLoadTexture2D(imagepath, 0);

    // Nice trilinear filtering.
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
    glGenerateMipmap(GL_TEXTURE_2D);

    // Return the ID of the texture we just created
    return textureID;
}
```

Note that the code above won't work in GLFW 3, but you get the idea. The function looked and worked more or less the same as our own BMP loading function, only it could be told to send the data to OpenGL straight away.

# Compressed Textures

At this point, you're probably wondering how to load JPEG files instead of TGA.

The short answer: don't. GPUs don't understand JPEG compression, so you'll compress your original image in JPEG, and decompress it so that the GPU can understand it. You're back to using uncompressed images in your program code, but you lost image quality while compressing to JPEG. If you have a *lot* of texture data, it makes sense to save disk space by compressing the files, but don't compress them too much, or you will notice the drop in image quality. A better choice for texture images is a format like PNG that uses non-destructive compression, and also supports a transparency channel (Alpha). To read and write JPEG and PNG images, you can use "libjpeg" and "libpng", which are freely available online.

There's also an option for using compressed texture formats natively on the GPU.

## Creating compressed textures

* Download the [AMD compress GUI](http://developer.amd.com/tools-and-sdks/graphics-development/amdcompress/), an AMD tool that is made available for free.
* Load a Power-Of-Two texture in it
* Generate mipmaps so that you won't have to do it in runtime
* Compress it in DXT1, DXT3 or DXT5 format (more about the differences between the various formats on [Wikipedia](http://en.wikipedia.org/wiki/S3_Texture_Compression))
* Export it as a .DDS file.

At this point, your image is compressed in a format that is directly compatible with the GPU. Whenever calling texture() in a shader, the GPU will uncompress it on-the-fly. This can seem slow, but since it takes a LOT less memory, less data needs to be transferred. Memory transfers are expensive, but the kind of simple texture decompression used by the DXT formats is essentially free (there is dedicated hardware for it). Typically, using texture compression yields a 20% increase in performance. So you save on performance and memory, but at the expense of a somewhat reduced quality.

## Using the compressed texture

Let's see how to load the image. It's very similar to the BMP code, except that the header is organized differently :

``` cpp
GLuint loadDDS(const char * imagepath){

    unsigned char header[124];

    FILE *fp;

    /* try to open the file */
    fp = fopen(imagepath, "rb");
    if (fp == NULL)
        return 0;

    /* verify the type of file */
    char filecode[4];
    fread(filecode, 1, 4, fp);
    if (strncmp(filecode, "DDS ", 4) != 0) {
        fclose(fp);
        return 0;
    }

    /* get the surface desc */
    fread(&header, 124, 1, fp); 

    unsigned int height      = *(unsigned int*)&(header[8 ]);
    unsigned int width         = *(unsigned int*)&(header[12]);
    unsigned int linearSize     = *(unsigned int*)&(header[16]);
    unsigned int mipMapCount = *(unsigned int*)&(header[24]);
    unsigned int fourCC      = *(unsigned int*)&(header[80]);
```

After the header is the actual data: all the pixels for all the mipmap levels. We can read them all in one batch :

``` cpp
    unsigned char * buffer;
    unsigned int bufsize;
    /* how big is it going to be including all mipmaps? */
    bufsize = mipMapCount > 1 ? linearSize * 2 : linearSize;
    buffer = (unsigned char*)malloc(bufsize * sizeof(unsigned char));
    fread(buffer, 1, bufsize, fp);
    /* close the file pointer */
    fclose(fp);
```

Here we'll deal with 3 different formats : DXT1, DXT3 and DXT5. We need to convert the "fourCC" flag into a value that OpenGL understands.

``` cpp
    unsigned int components  = (fourCC == FOURCC_DXT1) ? 3 : 4;
    unsigned int format;
    switch(fourCC)
    {
    case FOURCC_DXT1:
        format = GL_COMPRESSED_RGBA_S3TC_DXT1_EXT;
        break;
    case FOURCC_DXT3:
        format = GL_COMPRESSED_RGBA_S3TC_DXT3_EXT;
        break;
    case FOURCC_DXT5:
        format = GL_COMPRESSED_RGBA_S3TC_DXT5_EXT;
        break;
    default:
        free(buffer);
        return 0;
    }
```

Creating the texture is done as usual :

``` cpp
    // Create one OpenGL texture
    GLuint textureID;
    glGenTextures(1, &textureID);

    // "Bind" the newly created texture : all future texture functions will modify this texture
    glBindTexture(GL_TEXTURE_2D, textureID);
```

And now, we just have to fill each mipmap one after another :

``` cpp
    unsigned int blockSize = (format == GL_COMPRESSED_RGBA_S3TC_DXT1_EXT) ? 8 : 16;
    unsigned int offset = 0;

    /* load the mipmaps */
    for (unsigned int level = 0; level < mipMapCount && (width || height); ++level)
    {
        unsigned int size = ((width+3)/4)*((height+3)/4)*blockSize;
        glCompressedTexImage2D(GL_TEXTURE_2D, level, format, width, height, 
            0, size, buffer + offset);

        offset += size;
        width  /= 2;
        height /= 2;
    }
    free(buffer); 

    return textureID;
```

## Inversing the UVs

DXT compression comes from the DirectX world, where the V texture coordinate is flipped compared to OpenGL. So if you use compressed textures, you'll have to use (coord.u, 1.0-coord.v) to fetch the correct texel. You can do this flipping whenever you want: in your export script, in your loader or in your shader, but make sure to do it consistently in one place.

# Conclusion

You have just learnt to create, load and use textures in OpenGL.

In general, you should seriously consider using DXT compressed textures, since they are smaller to store, faster to load, and just as fast to use. The loss of quality is seldom noticeable, and the main drawback is that you have to pre-process your images with some compression tool. If you choose not to use DXT compression, use an image file format with non-destructive compression like PNG rather than JPEG. If you are only writing a small demo, feel free to use the BMP loading code presented above, or roll your own code for some other simple uncompressed file format like TGA, but both BMP and TGA are outdated and bad file formats that really deserve to be put to rest.

# Exercices

* The DDS loader is implemented in the source code, but not the texture coordinate modification. Change the code at the appropriate place to display the cube correctly.
* Experiment with the various DDS formats. They give different results, and different compression ratios.
* Try not generating mipmaps before saving the DXT file. What is the result? Think of 3 different ways to fix this.

# References

* [Using texture compression in OpenGL](http://www.oldunreal.com/editing/s3tc/ARB_texture_compression.pdf) , S&eacute;bastien Domine, NVIDIA
