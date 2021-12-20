---
layout: post
title: Vulkan Engine - Part 1
---

When it comes to computer science, computer graphics has always been one of my favorite topics. Initially I started off with playing around in OpenGL, but as soon as I discovered what Vulkan was, I had to try it.

Vulkan is a low-level graphics API developed by the Khronos group. It's a tough API to get down and is not recommended for beginners. However, once you get the basics down, it's extremely satisfying to watch the stuff being rendered by your program.

## Setting up development environment
Installing the Vulkan API is extremely simple and is slightly different depending on the operating system you're using. You can download your required version from the [LunarG website](https://vulkan.lunarg.com) and follow the instructions from there.

Next to the Vulkan SDK, we need a build tool! Lately I have been using Bazel instead of CMake as my primary build tool. After having to use it for work, it kind of stuck with me and now I like it more than CMake. There are some slight disadvantages to using Bazel as it's not as widely used as CMake, but follow along and I'll show you how to overcome those problems. In this blog I will only demonstrate how to integrate it with Bazel, but there are plenty other great guides describing how to build Vulkan-based applications with other build tools.

## Initializing the Bazel project
Now the fun begins! We can finally start developing our application. We need some small boilerplate in order to integrate the Vulkan SDK into our project, but don't worry, it's pretty simple!

So first, in the root folder of your project, create a file called `WORKSPACE` with the following content.
```python
# The name of our project.
workspace(
    name = "vulkan-engine"
)

# Load the bazel HTTP and GIT tools.
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")

# Clone the GIT repository containg the bazel rules for Vulkan.
git_repository(
    name = "rules_vulkan",
    remote = "https://github.com/jadarve/rules_vulkan.git",
    tag = "v0.0.3"
)

load("@rules_vulkan//vulkan:repositories.bzl", "vulkan_repositories")
vulkan_repositories("/path/to/where/you/installed/vulkan_sdk/")
```
Afterwards, create a file called `BUILD`, also in the root folder.
```python
config_setting (
    name = "linux",
    constraint_values = [
        "@platforms//os:linux"
    ],
    visibility = ["//visibility:public"]
)

config_setting (
    name = "windows",
    constraint_values = [
        "@platforms//os:windows"
    ],
    visibility = ["//visibility:public"]
)

config_setting (
    name = "macos",
    constraint_values = [
        "@platforms//os:macos"
    ],
    visibility = ["//visibility:public"]
)
```
The `BUILD` file contains several configuration settings for different operating systems as they each have a slightly different way of loading the Vulkan SDK.

Now we can start writing some code! First create a folder `main` and create a `BUILD` file there as well. The `BUILD` files are the same principle as `CMakeLists.txt` files and describes specific build rules for the contents of that folder. This file should have the following content.

```python
load("@rules_cc//cc:defs.bzl", "cc_binary")

cc_binary(
    name = "main",
    srcs = ["main.cc"],
    deps = select({
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
The `cc_binary` is what will create our (as the name already specifies) binary, in other words, the executable. All source files go in the `srcs` argument, headers in `hdrs` and dependencies in `deps`.

Now we can create our first source file to test that our Vulkan installation actually works. Create a file called `main.cc` in the `main` folder with the following contents.

```cpp
#include <iostream>
#include <string>
#include <vulkan/vulkan.hpp>

int main(int argc, char **argv) {
  const vk::ApplicationInfo appInfo = vk::ApplicationInfo()
                                          .setPApplicationName("vulkan-engine")
                                          .setApplicationVersion(0)
                                          .setEngineVersion(0)
                                          .setPEngineName("vulkan-engine");

  const vk::InstanceCreateInfo instanceInfo =
      vk::InstanceCreateInfo().setPApplicationInfo(&appInfo);

  vk::Instance instance;
  vk::Result result = vk::createInstance(&instanceInfo, nullptr, &instance);
  if (result != vk::Result::eSuccess) {
    std::cerr << "Failed to create Vulkan instance." << std::endl;
    exit(-1);
  }

  auto extensions = vk::enumerateInstanceExtensionProperties();
  for (const auto &ext : extensions) {
    std::cout << ext.extensionName << std::endl;
  }

  return EXIT_SUCCESS;
}
```

Try to build and run the project by running `bazel run //main:main` from the root folder of your project. The `bazel run` command automatically builds (and runs) the specified build target. You should get the following output (or something similar depending on the operating system you run it from).
```shell
> VK_KHR_device_group_creation
> VK_KHR_external_fence_capabilities
> VK_KHR_external_memory_capabilities
> VK_KHR_external_semaphore_capabilities
> VK_KHR_get_physical_device_properties2
> VK_KHR_get_surface_capabilities2
> VK_KHR_surface
> VK_EXT_debug_report
> VK_EXT_debug_utils
> VK_EXT_metal_surface
> VK_EXT_swapchain_colorspace
> VK_MVK_macos_surface
```
So what did we actually write? This is the easiest way of verifying your current setup is working. Basically, we create a Vulkan instance and ask it to print out its supported extensions. In the next part we will integrate this with GLFW.