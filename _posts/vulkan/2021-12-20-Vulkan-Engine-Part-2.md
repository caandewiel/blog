---
layout: post
title: Vulkan Engine - Part 2
category: vulkan
---

In the previous part we covered the basic setup of Vulkan and compiled our first Vulkan example using Bazel. In this second part we will connect Vulkan to [GLFW](https://www.glfw.org), an open source API which allows you to easily create windows and integrate them with Vulkan. There are alternatives (such as SDL), however I found that GLFW works well, is well supported and rather simple in use.

## Adding non-Bazel targets to Bazel project
One of the main drawbacks of Bazel is that it is not as widely supported as its alternatives (such as CMake). However, it is possible to add non-Bazel targets to your Bazel project. We need to write custom Bazel rules to do this. There are also alternatives we will explore in future parts to save some time here.

## Writing custom Bazel rules
Let's start by writing some custom Bazel rules. Since GLFW is a third party library, we will create a folder called `third_party`. In this folder, create a new build file called `glfw.BUILD`. For reference, these Bazel rules are based off [this](https://github.com/pbellis/bazel-glfw) and [this repository](https://github.com/siggb/glfw-bazel-xcode). I have altered them a little bit as they do not support mac OS.

```python
load("@rules_cc//cc:defs.bzl", "cc_library", "objc_library")

objc_library(
    name = "glfw_cocoa",
    hdrs = [
        "include/GLFW/glfw3.h",
        "include/GLFW/glfw3native.h",
        "src/cocoa_joystick.h",
        "src/nsgl_context.h",
        "src/cocoa_platform.h",
        "src/internal.h",
        "src/posix_thread.h",
        "src/egl_context.h",
        "src/osmesa_context.h",
        "src/mappings.h",
    ],
    srcs = [
        "src/cocoa_joystick.m",
        "src/nsgl_context.m",
        "src/cocoa_monitor.m",
        "src/cocoa_window.m",
        "src/cocoa_init.m",
        "src/posix_thread.c",
        "src/cocoa_time.c",
        "src/egl_context.c",
        "src/osmesa_context.c",
        "src/context.c",
        "src/init.c",
        "src/input.c",
        "src/vulkan.c",
        "src/window.c",
        "src/monitor.c",
    ],
    defines = [
        "_GLFW_COCOA",
    ],
    copts = [
        "-fno-objc-arc",
    ],
    visibility = [
        "//visibility:private",
    ],
)

cc_library(
    name = "glfw_src",
    srcs = [
        "src/context.c",
        "src/egl_context.c",
        "src/init.c",
        "src/input.c",
        "src/osmesa_context.c",
        "src/monitor.c",
        "src/vulkan.c",
        "src/window.c",
        "src/xkb_unicode.c",
    ] + select({
        "@platforms//os:windows": [
            "src/win32_init.c",
            "src/win32_joystick.c",
            "src/win32_monitor.c",
            "src/win32_thread.c",
            "src/win32_time.c",
            "src/win32_window.c",
            "src/wgl_context.c",
        ],
        "@platforms//os:macos": [
            "src/cocoa_time.c",
            "src/posix_thread.c",
        ],
        "//conditions:default": [
            "src/glx_context.c",
            "src/linux_joystick.c",
            "src/posix_thread.c",
            "src/posix_time.c",
            "src/x11_init.c",
            "src/x11_monitor.c",
            "src/x11_window.c",
        ],
    }),
    hdrs = [
        "include/GLFW/glfw3.h",
        "include/GLFW/glfw3native.h",
        "src/egl_context.h",
        "src/internal.h",
        "src/osmesa_context.h",
        "src/mappings.h",
        "src/xkb_unicode.h"
    ] + select({
        "@platforms//os:windows": [
            "src/win32_joystick.h",
            "src/win32_platform.h",
            "src/wgl_context.h", 
        ],
        "@platforms//os:macos": [
            "src/cocoa_joystick.h",
            "src/cocoa_platform.h",
            "src/glx_context.h",
            "src/nsgl_context.h",
            "src/null_joystick.h",
            "src/null_platform.h",
            "src/posix_thread.h",
            "src/wl_platform.h",
        ],
        "//conditions:default": [
            "src/glx_context.h",
            "src/linux_joystick.h",
            "src/posix_thread.h",
            "src/posix_time.h",
            "src/x11_platform.h",
        ],
    }),
    defines = select({
        "@platforms//os:windows": [
            "_GLFW_WIN32",
        ],
        "@platforms//os:macos": [
            "_GLFW_COCOA",
            "_GLFW_NSGL",
            "_GLFW_NO_DLOAD_WINMM",
            "_GLFW_USE_OPENGL",
        ],
        "//conditions:default": [
            "_GLFW_HAS_XF86VM",
            "_GLFW_X11",
        ],
    }),
    linkopts = select({
        "@platforms//os:windows": [
            "-DEFAULTLIB:user32.lib",
            "-DEFAULTLIB:gdi32.lib",
            "-DEFAULTLIB:shell32.lib",
        ],
        "@platforms//os:macos": [
            "-framework OpenGL",
            "-framework Cocoa",
            "-framework IOKit",
            "-framework CoreFoundation",
        ],
        "//conditions:default": [],
    }),
    deps = select({
        "@platforms//os:macos": [
            ":glfw_cocoa",
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "glfw",
    hdrs = [
        "include/GLFW/glfw3.h",
        "include/GLFW/glfw3native.h",
    ],
    deps = [
        ":glfw_src",
    ],
    strip_include_prefix = "include",
    visibility = ["//visibility:public"],
)
```

I will not go into too much detail as to what every line does exactly. Basically we just create a `cc_library` which creates the static library for GLFW. If any of these keywords are confusing, I would recommend to take a look at the [Bazel documentation](!https://docs.bazel.build/versions/main/be/c-cpp.html#cc_library). The rules for GLFW are rather complex as they are very OS dependent, so don't worry if you do not fully understand everything that's happening here. Just make sure you understand the bigger picture.

Afterwards we will use the `new_git_repository` functionality from Bazel to automatically clone the repository and inject our custom BUILD file to build GLFW. First we need to include `new_git_repository` in our `WORKSPACE` file.

```python
load("@bazel_tools//tools/build_defs/repo:git.bzl", "new_git_repository")
```

Now we can define our target for GLFW. When we use `new_git_repository`, we need to provide it with the repository, the commit to use and the custom `BUILD` file.

```python
new_git_repository(
    name = "glfw",
    remote = "https://github.com/glfw/glfw.git",
    commit = "7d5a16ce714f0b5f4efa3262de22e4d948851525",
    build_file = "//third_party:glfw.BUILD",
)
```

What this will do for us, is create a new target `@glfw//...` where the `...` can be replaced with anything defined in the custom `BUILD` file. In our case, that is a `cc_library` called `glfw`, so our full target path will be `@glfw//:glfw`. Now we can add it to the dependencies of `//main:main`.

```python
cc_binary(
    name = "main",
    srcs = ["main.cc"],
    deps = [
        "@glfw//:glfw",
    ] + select({
        "//:windows": [
            "@vulkan_windows//:vulkan_cc_library",
        ],
        "//:macos": [
            "@vulkan_macos//:vulkan_cc_library",
        ],
        "//conditions:default": [],
    }),
)
```

Sweet right? This is where the main advantage of Bazel comes to light. It's significantly more intuitive to use than CMake (in my opinion of course). Nowe we have succesfully added GLFW to our project, we can use it in our C++ files as well.

First let is include the GLFW header.
```cpp
#include <GLFW/glfw3.h>

int main(int argc, char **argv) {
  glfwInit();
  auto *window = glfwCreateWindow(1280, 720, "Vulkan", nullptr, nullptr);

  // ... Code from previous part here.

  while (!glfwWindowShouldClose(window)) {
    glfwPollEvents();
  }

  glfwDestroyWindow(window);
  glfwTerminate();

  return EXIT_SUCCESS;
}
```

You should be greeted with a nice window looking something like this.

![Window created through GLFW built using Bazel.]({{ site.baseurl }}/images/Vulkan-Engine/glfw_window.png)