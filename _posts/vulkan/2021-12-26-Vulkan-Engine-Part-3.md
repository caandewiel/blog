---
layout: post
title: Vulkan Engine - Part 3
category: vulkan
---
In this part we will create our own abstracted layer on top of the Vulkan instance. We will extend this with support for Vulkan validation layers which will help us tremendously when debugging.

## Vulkan Instance abstraction

### `Instance` Header

First create a new header file, called `engine/core/instance.h`. Here we will create a high level class for a Vulkan instance which we will later extend with some debugging tools. Since Vulkan is a C-like API, we also need to make sure unused resources are properly cleaned up when possible. `vulkan.hpp` does support some automatic cleanup features similar to `std::unique_ptr`, however I found that these are not always working as they should so I prefer to free these objects explicitly and the destructor of our abstraction class is ideal for this task.

```cpp
namespace engine::core {
  constexpr auto validationLayers = {"VK_LAYER_KHRONOS_validation"};

  class Instance {
   public:
    Instance();
    ~Instance();

    [[nodiscard]] const vk::Instance &getInstance() const;

   private:
    vk::Instance m_instance;
  };
}  // namespace engine::core
```

### Required Extensions
Now we can start creating the implementation. Create a file called `engine/core/instance.cc`. Here we will first define a function `getRequiredExtensions()` in an anonymous namespace, as this will be the only location in the codebase where it will be used. It's purpose is to initialize a `std::vector` containing all extensions which are required for our vulkan instance. For now it will only contain the GLFW extensions, but we'll extend that later this part to include the debug extensions as well.

```cpp
namespace {
std::vector<const char *> getRequiredExtensions() {
  uint32_t glfwExtensionCount = 0;
  const auto **glfwExtensions =
      glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

  return {glfwExtensions, glfwExtensions + glfwExtensionCount};
}
}  // namespace
```

### `Instance` Implementation
After this, we can start implementing our `Instance` class. We need a constructor `Instance()`, destructor `~Instance()` and a function to get an immutable version of the `vk::Instance` called `getInstance()`.

Vulkan is a very verbose API requiring a lot of boilerplate which might be a bit overwhelming. Vulkan objects are usually created using a `vk::SomeObjectCreateInfo` struct containing all the necessary attributes.

For now, we will only need a `vk::ApplicationInfo` and a `vk::InstanceCreateInfo` object and luckily all their attributes are rather self explanatory, so I won't go into too much detail here.

Most Vulkan objects need to be explicitly freed when not necessary anymore by calling the `.destroy()` function (or `vkDestroySomeObject()` when using `vulkan.h` instead of `vulkan.hpp`). I highly recommend using `vulkan.hpp` here as it is way easier and cleaner to use.
```cpp
namespace engine::core {
  Instance::Instance() {
    vk::ApplicationInfo applicationInfo;
    applicationInfo.setPApplicationName("Vulkan Engine");
    applicationInfo.setApplicationVersion(1);
    applicationInfo.setPEngineName("Vulkan Engine");
    applicationInfo.setEngineVersion(1);
    applicationInfo.setApiVersion(VK_API_VERSION_1_2);

    const auto extensions = getRequiredExtensions();

    vk::InstanceCreateInfo instanceCreateInfo;
    instanceCreateInfo.setFlags(vk::InstanceCreateFlags());
    instanceCreateInfo.setPApplicationInfo(&applicationInfo);
    instanceCreateInfo.setPEnabledExtensionNames(extensions);
    instanceCreateInfo.setPEnabledLayerNames(validationLayers);
  }

  Instance::~Instance() { m_instance.destroy(); }

  const vk::Instance &Instance::getInstance() const { return m_instance; }
}  // namespace engine::core
```

### `Instance` Build File
Finally, create a file called `engine/core/BUILD`. Since we use GLFW to fetch our required extensions, we need to add it as dependency to `//engine/core:instance`.

```python
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "instance",
    srcs = [
        "instance.cc",
    ],
    hdrs = [
        "instance.h",
    ],
    visibility = ["//visibility:public"],
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

### Use our `Instance` class
Now we can create a Vulkan instance through our abstraction class! Even better, when it gets out of scope, it will automatically call the Vulkan API to clean up the instance as it's not needed anymore. Do remember to add `//engine/core:instance` as dependency in `main/BUILD` for target `main`.
```cpp
#include "engine/core/instance.h"

int main(int argc, char **argv)
{
  // Code from previous part.
  engine::core::Instance instance;
}
```
Now compile and test it by running `bazel run //main:main` and your program should run just fine! Now we can start adding the validation layers.

## Vulkan Validation Layers
We can now start extending our `Instance` class so it is able to use the validation layers from Vulkan. Validation layers are extremely useful when it comes to debugging your Vulkan program. Later in this part we'll check out some of these validation layers.

### Debug Utils Callback
Before we do anything, we need to register the debug utils extension to the Vulkan instance. We can do this by altering the `getRequiredExtensions()` function a bit. It should look something like this.

```cpp
std::vector<const char *> getRequiredExtensions() {
  uint32_t glfwExtensionCount = 0;
  const auto **glfwExtensions =
      glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

  std::vector<const char *> extensions = {glfwExtensions, glfwExtensions + glfwExtensionCount};
  extensions.emplace_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);

  return extensions;
}
```

When the Vulkan validation layers encounter any problem, they will report it by sending a `VkDebugUtilsMessengerCallbackDataEXT` to a registered callback function. Let's define this callback function first, we can add it to the same anonymous namespace as `getRequiredExtensions()`. This callback is based off the one mentioned in [this great resource by Alexander Overoorde](https://vulkan-tutorial.com/code/04_logical_device.cpp). Basically it takes a `VkDebugUtilsMessengerCallbackDataEXT` object and it prints out its data.

```cpp
static VKAPI_ATTR VkBool32 VKAPI_CALL
debugCallback(VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
              VkDebugUtilsMessageTypeFlagsEXT messageType,
              const VkDebugUtilsMessengerCallbackDataEXT *pCallbackData,
              void *pUserData) {
  std::cerr << "Validation Layer: " << pCallbackData->pMessage << std::endl;

  return VK_FALSE;
}
```

### `DebugUtilsMessenger` Header
The debug utils from Vulkan might seem a little bit hacky at first as they are not really designed for C++. I love to have code that automatically frees up objects when they are not necessary anymore, but found this to be a bit challenging with the Vulkan debug utils, but it is possible. 

Let's start by adding 2 new private attributes to our `Instance` class.

```cpp
vk::DebugUtilsMessengerEXT m_debugUtilsMessenger;
```

Now we can initialize our `vk::DebugUtilsMessengerEXT`. You might have already guessed, we have to do this by creating a `vk::DebugUtilsMessengerCreateInfoEXT`. Here we specify the callback function `debugCallback`, the severity and types of messages we want to report to our callback.

```cpp
vk::DebugUtilsMessengerCreateInfoEXT debugUtilsMessengerCreateInfo;

debugUtilsMessengerCreateInfo.setMessageSeverity(
    vk::DebugUtilsMessageSeverityFlagBitsEXT::eVerbose |
    vk::DebugUtilsMessageSeverityFlagBitsEXT::eWarning |
    vk::DebugUtilsMessageSeverityFlagBitsEXT::eError);
debugUtilsMessengerCreateInfo.setMessageType(
    vk::DebugUtilsMessageTypeFlagBitsEXT::eGeneral |
    vk::DebugUtilsMessageTypeFlagBitsEXT::eValidation |
    vk::DebugUtilsMessageTypeFlagBitsEXT::ePerformance);
debugUtilsMessengerCreateInfo.setPfnUserCallback(debugCallback);
```

The `vk::DebugUtilsMessengerCreateInfo` needs to be passed to the `setPNext()` function on the `vk::InstanceCreateInfo` object. As described by the [Vulkan documentation](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstanceCreateInfo.html), `pNext` is a pointer to a structure extending the current structure.

```cpp
instanceCreateInfo.setPNext(&debugUtilsMessengerCreateInfo);
```

### Automatically free up `vk::DebugUtilsMessengerEXT`
I found this to be a little bit messy in `vulkan.hpp` but I managed to find a way to properly free up `vk::DebugUtilsMessengerEXT`. The main problem here is that the functions to create and destroy the `vk::DebugUtilsMessengerEXT` are not by default loaded in the Vulkan SDK. Since we use `vulkan.hpp`, we can use `DispatchLoaderDynamic` implementation to load all function pointers known (more info [here](https://github.com/KhronosGroup/Vulkan-Hpp#extensions--per-device-function-pointers)). It's a bit of a messy process, so pay some extra attention here (and don't worry if you don't fully understand what's going on here, it's the only location where this is necessary)!

First we need to define `VULKAN_HPP_DISPATCH_LOADER_DYNAMIC`. Add the following to your `engine/core/instance.h`.
```cpp
#define VULKAN_HPP_DISPATCH_LOADER_DYNAMIC 1
#include <vulkan/vulkan.hpp>
```

Now we only need to provide storage for the default `DispatchLoader`. Add the following to `engine/core/ince.cpp`.

```cpp
#include "engine/core/instance.h"

#include <GLFW/glfw3.h>

VULKAN_HPP_DEFAULT_DISPATCH_LOADER_DYNAMIC_STORAGE
```

Now we can use the `VULKAN_HPP_DEFAULT_DISPATCHER` macro to use this dispatcher. So now, add the following code to the end of the constructor `Instance()`.
```cpp
  vk::DynamicLoader dl;
  PFN_vkGetInstanceProcAddr vkGetInstanceProcAddr =
      dl.getProcAddress<PFN_vkGetInstanceProcAddr>("vkGetInstanceProcAddr");
  VULKAN_HPP_DEFAULT_DISPATCHER.init(vkGetInstanceProcAddr);
  m_instance = vk::createInstance(instanceCreateInfo, nullptr);
  VULKAN_HPP_DEFAULT_DISPATCHER.init(m_instance);
    m_debugUtilsMessenger = m_instance.createDebugUtilsMessengerEXT(
        debugUtilsMessengerCreateInfo, nullptr, VULKAN_HPP_DEFAULT_DISPATCHER);
```

Let's try to run our code so far and see whether our validation layers actually work! If you run it, you should get the following output.

```shell
> Validation Layer: Validation Error: [ VUID-vkDestroyInstance-instance-00629 ] Object 0: handle = 0x126008200, type = VK_OBJECT_TYPE_INSTANCE; Object 1: handle = 0x10000000001, type = VK_OBJECT_TYPE_DEBUG_UTILS_MESSENGER_EXT; | MessageID = 0x8b3d8e18 | OBJ ERROR : For VkInstance 0x126008200[], VkDebugUtilsMessengerEXT 0x10000000001[] has not been destroyed. The Vulkan spec states: All child objects created using instance must have been destroyed prior to destroying instance (https://vulkan.lunarg.com/doc/view/1.2.170.0/mac/1.2-extensions/vkspec.html#VUID-vkDestroyInstance-instance-00629)
> Validation Layer: Validation Error: [ VUID-vkDestroyInstance-instance-00629 ] Object 0: handle = 0x126008200, type = VK_OBJECT_TYPE_INSTANCE; Object 1: handle = 0x10000000001, type = VK_OBJECT_TYPE_DEBUG_UTILS_MESSENGER_EXT; | MessageID = 0x8b3d8e18 | OBJ ERROR : For VkInstance 0x126008200[], VkDebugUtilsMessengerEXT 0x10000000001[] has not been destroyed. The Vulkan spec states: All child objects created using instance must have been destroyed prior to destroying instance (https://vulkan.lunarg.com/doc/view/1.2.170.0/mac/1.2-extensions/vkspec.html#VUID-vkDestroyInstance-instance-00629)
> Validation Layer: Unloading layer library /usr/local/share/vulkan/explicit_layer.d/../../../lib/libVkLayer_khronos_validation.dylib
> Validation Layer: Unloading layer library /usr/local/share/vulkan/explicit_layer.d/../../../lib/libVkLayer_khronos_validation.dylib
```

If we take a close look, we see the following output coming from our validation layers.
```shell
VkDebugUtilsMessengerEXT 0x10000000001[] has not been destroyed.
```
This means that our validation layers work! They warn us that we did not free up our `vk::DebugUtilsMessengerEXT` before destroying our `vk::Instance`. Simple fix right?! Just change your destructor and you should have no errors anymore!

```cpp
Instance::~Instance() {
    m_instance.destroyDebugUtilsMessengerEXT(m_debugUtilsMessenger, nullptr, VULKAN_HPP_DEFAULT_DISPATCHER);
    m_instance.destroy();
}
```