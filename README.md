# Building and Linking Google's ANGLE with Raylib on MacOS

**Table of Contents**
[TOC]

## 1. Introduction

If there's one thing I like more than graphics, it' fast graphics...

By the end of this post, you'll know you how to build and link Google's ANGLE with Raylib, giving at least a 3x boost in raw frame-rate on Mac. In fact, its the same low-level graphics library that powers Google Chrome! (And any chromium based browsers)

This tutorial assumes some basic knowledge of compiling C/C++ software and some comfort in the command line.

You'll also need the followng prerequisites:


1. Apple Developer Command Line Tools (required)
2. VS Code (recommended)
3. Homebrew (recommended)
4. Ninja `brew install ninja`
5. CMake `brew install cmake`
4. Python 3 (required)
5. Git (required)


> The impatient can skip to the [Short Version](#3-quickstart-short-version-using-angle-from-your-browser), showing your how to heist `libEGL.dylib` and `libGLESv2.dylib` from Chrome.

> If you'd like a more complete guide, and want to learn how to build raylib and ANGLE from scratch and tie it together in a nice cmake project, please checkout the [Slightly Longer Version](#4-slightly-longer-version)



## 2. Why use ANGLE anyway?

It seem's everyone's got their own graphics API these days. Microsoft sees the future in Direct3D. Kronos (developer of the OpenGL and Vulkan specs) sees Vulkan as the next-gen solution. Apple on the other hand, true to their nature of building fullstack, hardware and software projects, choose freedom and flexibility and decided they too needed their own shiny accelerated graphics API - Metal
(NVidia supports as many as it can, they just want to sell hardware.)

Now usually, OpenGL has a pretty good cross-platform support. And thats usually the go-to. But with since 2018, Apple basically deprecating OpenGL support in favor of pursuring Metal which can get about 10x more draw calls in the same time as OpenGL on modern Apple processors, specfically the ARM (M1 and M2) processors.

With all these different APIs, abstraction layers are starting to popup. In fact, Google develops such an API thats present in all Chromium based browsers called ANGLE. It provides a compliant OpenGL ES 2.0 frontend with backends to all the major native graphics APIs. 

> In short, you can get HUGE speed boosts when using Metal through ANGLE on MacOS platforms. Usually double, tripple, or even more the framerate of OpenGL powered solutions.

## 3. Quickstart (Short Version Using ANGLE from your Browser)

Shout out to [Peter0x44](https://github.com/Peter0x44) on github for point this out, but you can actually just grab ANGLE in the form of `libEGL.dylib` and `libGLESv2.dylib` from a Chromium based browser your probably already have!

Here we go.

**Step 1: Hiest `libEGL.dylib` and `libGLESv2.dylib` from Chrome**

Using Chrome for example, lets dig down into the `.app` and grab these files. We'll use the `find` command:

```shell
cd /Applications/Google\ Chrome.app
find ./ -name 'libEGL.dylib'
```


Your output should look something like this. Sweet.

```
.//Contents/Frameworks/Google Chrome Framework.framework/Versions/113.0.5672.126/Libraries/libEGL.dylib
.//Contents/Frameworks/Google Chrome Framework.framework/Versions/114.0.5735.90/Libraries/libEGL.dylib
```

You can now locate `libGLESv2.dylib`:

```
find ./ -name 'libGLESv2.dylib'
```

Boom:

```
.//Contents/Frameworks/Google Chrome Framework.framework/Versions/113.0.5672.126/Libraries/libGLESv2.dylib
.//Contents/Frameworks/Google Chrome Framework.framework/Versions/114.0.5735.90/Libraries/libGLESv2.dylib
```

**Step 2: Copy `libEGL.dylib` and `libGLESv2.dylib` raylib to your project folder**

```
cd <your project>
cp "/Applications/Google Chrome.app/Frameworks/Google Chrome Framework.framework/Versions/114.0.5735.90/Libraries/libGLESv2.dylib" ./
cp "/Applications/Google Chrome.app/Contents/Frameworks/Google Chrome Framework.framework/Versions/114.0.5735.90/Libraries/libEGL.dylib" ./
```

**Step 3: Link with raylib**

Here's a one-liner if you're using a static build of raylib:
```bash
g++ -std=c++14 -I/path/to/raylib/include -o main main.cpp /path/to/raylib.a /path/to/libEGL.dylib /path/to/libGLESv2.dylib
```

## 4. Slightly Longer Version

A single `g++` or `gcc` command is fine for small projects, but for most projects, you're going to want a build system. We'll go with CMake since it seems to be the defactor standard for C/C++ projects.

### 4.1 Folder Structure & Setup

**Step 1: Folder setup**

Go ahead and pick a root project on your compute. I suggest something like `~/code/raylib-angle` or something easy.

Put the following in the folder:

```
.
├── CMakeLists.txt
├── build
├── src
│   └── main.cpp
└── vendor
    ├── angle
    ├── raygui
    └── raylib
```

### 4.2 Git submodules

While you can certainly link with raylib static builds, I like to keep everything bundled as source code you can compile without fuss. So lets add `raylib` and `raygui` if you're feeling adventerous.

**Add raylib as a submodule to `vender/raylib`**

```
git submodule add https://github.com/raysan5/raylib.git vendor/raylib
```

**Add raylib as a submodule to `vendor/raygui`**

```
git submodule add https://github.com/raysan5/raygui.git vendor/raygui
```

**Add ANGLE as a submodule to `vendor/angle`**

```
git submodule add https://chromium.googlesource.com/angle/angle vendor/angle
```

**Initialize the submodules**
You only really need to do this if you're cloning your root repo, but as a reminder for the future run:

```
git submodule update --init --recursive
```

Cool, now lets build ANGLE

### 4.2 Building Angle

I haven't quite got this tied into my `CMakeLists.txt` yet since angle uses a seperate build system, `ninja` (which is crazy fast btw), so we'll build ANGLE seperately for now.

**Step 1: Bootstrap ANGLE**

```
cd vendor/angle
python3 scripts/bootstrap.py
gclient sync
```

**Step 2: Generate Ninja Build Files**

```
gn gen out/Release --args='is_debug=false'
```

Here we generated build files for release (optimized) build, but you can generate a Debug build my removing `--args='is_debug=false' and running `gn gen out/Debug`

**Step 3: Compile Angle**

This step will take a hot minute. It took about 10 mins on my 2021 Macbook M1 Pro. The debug build was about half that time.

```
ninja -j 10 -k1 -C out/Release
```

Here I used all 10 CPUs. Use something appropriate for your system.

When complete, the shared librarys (dylib's) you want are `libEGL.dylib` and `libGLESv2.dylib` found in the `vendor/angle/out/Release/` copy those too your project root: `~/code/raylib-angle` (or whatever yours is)

### 4.3 Building the Project

Now lets setup a quick `CMakeLists.txt` and compile raylib and its bundled code like `glfw`

**Step 1: Create CMakeLists.txt**

Add this to `<project-root>/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.16)

project(raylib-starter)

set(CMAKE_CXX_STANDARD 14)

# Add raylib subdir
add_subdirectory(vendor/raylib)

# Add my sources
file(GLOB_RECURSE SOURCES "src/*.cpp")
add_executable(${PROJECT_NAME} ${SOURCES})

# Define path to ANGLE libraries
set(ANGLE_LIBRARY_DIR vendor/angle/out/Release)

# Find ANGLE libraries
find_library(ANGLE_GLESv2_LIBRARY libGLESv2.dylib PATHS ${ANGLE_LIBRARY_DIR})
find_library(ANGLE_EGL_LIBRARY libEGL.dylib PATHS ${ANGLE_LIBRARY_DIR})

# Add raylib include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/raylib/src)

# Add raygui include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/raygui/src)

# Add ANGLE include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/angle/include)

# Link against raylib and ANGLE libraries
target_link_libraries(${PROJECT_NAME} raylib ${ANGLE_EGL_LIBRARY} ${ANGLE_GLESv2_LIBRARY})

```

**Step 2: Create `<project-root>src/main.cpp`**

Here's some hello world raylib code:

```cpp
#include "raylib.h"

int main(void)
{

    const int screenWidth = 800;
    const int screenHeight = 450;

    InitWindow(screenWidth, screenHeight, "raylib [core] example - basic window");

    SetTargetFPS(60); 
    
    while (!WindowShouldClose()) 
    {
        BeginDrawing();
            ClearBackground(RAYWHITE);
            DrawText("Congrats! You created your first window!", 190, 200, 20, LIGHTGRAY);

            DrawFPS();
        EndDrawing();
    }
    CloseWindow();
    return 0;
}
```


### 4.4 Performance Testing
### 5. Raylib cmake-starter (shamless plug)

I recently discovered the amazing raylib ecosystem by Ramon Santiago (Github: raysan5) & Contributors. Raylib is a high performance, lightweight, self-contained, and extremely portable 2D/3D graphics library with easy input, audio, textures, fonts, 3D models, cameras, and more with 60+ language bindings.


The primary motivation behind Raylib, was to make game programming fun and easy. Raylib isn't limited to games either. It doesn't impose a rigid engine or system. It's like the Python Flask of the graphics world, but with a few more optional batteries.

In fact, I use raylib mostly for high-performance analytics visualization tools for a 5G/LTE Network Simulator I build for my day job, drawing hardware accelerated graphs, debuggers, and tools is a breeze.

Enough with the evangelizing, let's get to work.

## Why?




## Whats in the Box?

1. Quick-start with raylib and CMake
2. Build and link Google ANGLE for huge speed boosts on Mac's with M1/M2 Chips.
3. Shortcut and get ANGLE from your browser and link it.


## Prerequisites

While I'll try to be thorough, but I'll assume some familiarity with C/++, the basics of compiling software, and that you have the following tools:

1. VS Code (recommended)
2. Apple Developer Command Line Tools (required)
3. Homebrew (recommended)
4. Ninja `brew install ninja`
5. CMake `brew install cmake`
4. Python 3 (required)
5. Git (required)




## Getting Setting Up Submodules


First, we'll add `raylib`, `raygui`, and `angle` as submodules.

**Raylib**



**Raygui**

This stage is optional, but its nice to get something useable on the screen.

```
git submodule add https://github.com/raysan5/raylib.git vendor/raylib
```

**ANGLE**

```
git submodule add https://chromium.googlesource.com/angle/angle vendor/angle
```

**Initialize Submodules**

```

```

## Building ANGLE

We're using `cmake` for our build, but ANGLE, as a Google project, uses Googles ninja build system.

Luckily, ANGLE comes with a bootstrapping script which graphs all of angle's dependancies.

**Step 1: Bootstrap ANGLE**

```
cd vendor/angle
python3 scripts/bootstrap.py
gclient sync
```

**Step 2: Generate Ninja Build Files**

```
gn gen out/Release --args='is_debug=false'
```

Here we generated build files for release (optimized) build, but you can generate a Debug build my removing `--args='is_debug=false' and running `gn gen out/Debug`

**Step 3: Compile Angle**

This step will take a while. It took about 10 mins on my 2021 Macbook M1 Pro. The debug build was about half that time.

```
ninja -j 10 -k1 -C out/Release
```

Here I used all 10 CPUs. Use something appropriate for your system.

When complete, the shared librarys (dylib's) you want are `libEGL.dylib` and `libGLESv2.dylib` found in the `vendor/angle/out/Release/` folder.

### CMakeLists.txt

In the root of your project, add a CMakeLists.txt file with the following contents. You can certainly compile and link simple projects with a single `gcc` command, but cmake makes modularly adding other third party code a bit nicer.

```
cmake_minimum_required(VERSION 3.16)

project(raylib-angle)

set(CMAKE_CXX_STANDARD 14)

# Add the raylib submodule
add_subdirectory(vendor/raylib)

# Add out sources
file(GLOB_RECURSE SOURCES "src/*.cpp")
add_executable(${PROJECT_NAME} ${SOURCES})

# Define path to ANGLE libraries
set(ANGLE_LIBRARY_DIR vendor/angle/out/Release)

# Find ANGLE libraries
find_library(ANGLE_GLESv2_LIBRARY libGLESv2.dylib PATHS ${ANGLE_LIBRARY_DIR})
find_library(ANGLE_EGL_LIBRARY libEGL.dylib PATHS ${ANGLE_LIBRARY_DIR})

# Add raylib include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/raylib/src)

# Add raygui include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/raygui/src)

# Add ANGLE include directory
target_include_directories(${PROJECT_NAME} PRIVATE vendor/angle/include)

# Link against raylib and ANGLE libraries
target_link_libraries(${PROJECT_NAME} raylib ${ANGLE_EGL_LIBRARY} ${ANGLE_GLESv2_LIBRARY})

```

### src/main.cpp

Here we'll setup and the standard hello world for raylib



## Building the Project

Now the fun part, lets build with cmake!

**Step 1: Generate Build files**

```
cmake -DCUSTOMIZE_BUILD=ON -DOPENGL_VERSION="ES 2.0" -B build
```

The `CUSTOMIZE_BUILD=ON` option is a raylib cmake option which allows some build customizations, and `-DOPENGL_VERSION="ES 2.0"` tell's raylib we want to use OpenGL ES 2.0 which is what will cause it to look and use the ANGLE libraries we've built.

**Step 2: Build**

Now we can run the actual build. This could be pretty quick since it's just building raylib, raylib (if you choose to add that), and bundled dependancies like `glfw`

```
cmake --build build
```

**Troubleshooting**

If you get an error like this:

```
error: incompatible pointer types assigning to 'int *' from 'bool *'
    if (active == NULL) active = &temp;
```

Just edit this line in `vendor/raygui/src/raygui.h` and change these lines:

```
    bool temp = false;
    if (active == NULL) active = &temp;
```

to

```
    bool temp = false;
    if (active == NULL) active = (int *)&temp;
```

This casts `temp`, which is a `bool *` to an `int *`

You could also change `bool temp` to `int temp`

> I'll submit a pull request for this fix later. This may not be an issue by the time you read this.