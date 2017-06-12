---
layout: post
title:  "Developing a Software Renderer Part 3"
date:   2017-06-12 15:14:00 +0200
#last_modified_at: 2017-06-10 10:10:00 +0200
feature_image: "https://unsplash.it/1200/400?image=41"
categories: [Software Rendering]
tags: [rasterization, rendering, c++]
---
In this post I describe how to add pixel shader capabilities to the software
rasterizer and how to optimize it even further for example using OpenMP to
parallelize the rasterization.

<!-- more -->

## Pixel Shader Support

In the current code the per pixel operations are coded directly inside the
innermost loop. We want to have the ability to do arbitrary work per pixel and
use the rasterizer as a library.

One simple way to do this would be to define an interface `IPixelShader` with a
virtual `drawPixel` method and pass this to the rasterizer. I already tried this
out and found out that the virtual function call is way to expensive and slows
everything down too  much. The second best option would be a function pointer
but I assumed this will be slow too.

The best thing would be if the compiler could inline our per pixel code, so that
there is no function call at all. It turns out there is such a possibility with
C++ templates.

We can parameterize the whole triangle rasterization method with our pixel
shader class using a c++ template and call a static method in the pixel shader
class to draw our pixels. This way the compiler can inline the pixel drawing
method inside the inner loop and this should give the best performance.

The pixel shader class will then look like this:

```cpp
```

And we can set the pixel shader and then draw like this:

```cpp
rasterizer.setPixelShader<PixelShader>();
rasterizer.drawTriangle(v0, v1, v2);
```

The trick here is that the `setPixelShader<PixelShader>()` method gets a member
function pointer to the actual `drawTriangleTemplate<PixelShader>` function and
stores it in a variable so that the `drawTriangle` call does not require the
template any longer and will just forward the call to the stored member
function.

[Part 2]({% post_url 2017-06-10-developing-a-software-renderer-part2 %}

## Parallelization Using OpenMP

With OpenMP we can parallelize our software rasterizer in a simple way.
We can use some `#pragma` statements to do this.

To use OpenMP with CMake we can use the following code in the CMakeLists.txt
file:

```cmake
find_package(OpenMP)
if (OPENMP_FOUND)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif ()
```

In the rasterizer we can parallelize the for loop which iterates over all the
blocks that need to be drawn. We could prepend the following pragma and be done:

```cpp
#pragma omp parallel for collapse(2)
```

Unfortunately the `collapse` only works on Linux since there we have OpenMP 3.0.
Under Windows with Visual Studio we need a workaround. We need to restructure
the loop.

```cpp
int stepsX = (maxX - minX) / BlockSize + 1;
int stepsY = (maxY - minY) / BlockSize + 1;

#pragma omp parallel for
for (int i = 0; i < stepsX * stepsY; ++i)
{
  int sx = i % stepsX;
  int sy = i / stepsX;
  
  int x = minX + sx * BlockSize;
  int y = minY + sy * BlockSize;

  // Add 0.5 to sample at pixel centers.

  float xf = x + 0.5f;
  float yf = y + 0.5f;

  [...]
```

When we now run the rasterizer it is indeed faster but I only achieved a speedup
of 2x.