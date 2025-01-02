---
#date: '2025-01-01T15:34:31+01:00'
tags:
  - vulkan
  - c++
  - Win32
  - X11
weight: 2
categories:
  - projects
author: Matej Elias
draft: false
title: "VulkanSurfer - minimal window library for Vulkan"
ShowToc: true
TocOpen: true
hidemeta: false
comments: true
description: 'A minimal cross-platform object-oriented c++11 header only window library for Vulkan.'
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
#ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: false
editPost:
    URL: "https://github.com/elliahu/VulkanSurfer"
    Text: "Source code" # edit text
---
## About this project
In graphics engines and renderers, window management and event handling is often a necessary but tedious task. Most developers choose to abstract this functionality using libraries like GLFW, SDL, or similar tools. While these libraries are robust and feature-rich, they tend to be heavyweight and require installation via NuGet, vcpkg, or manual downloads. This can impact the portability of projects that rely on them.

As a researcher and graphics programming enthusiast, my needs are simpler. I primarily need a window for the rendering surface and basic input handling—just enough to close the window with the Esc key or enable basic camera movement. For such straightforward requirements, using a large library often feels like overkill. Don’t get me wrong—SDL2 is excellent, and GLFW is truly amazing. But for smaller projects, their size and complexity can be unnecessary.

When I looked for lightweight, single-header alternatives, I was surprised to find a lack of options tailored to Vulkan. The OpenGL ecosystem has plenty of solutions, but Vulkan seems to have been overlooked in this area. That’s why I decided to create my own solution.

What started as a utility extracted from my renderer has evolved into a portable, single-header window library designed specifically for Vulkan. It’s simple, lightweight, and focuses on exactly what’s needed—nothing more, nothing less.

This [project](https://github.com/elliahu/VulkanSurfer) aims to make creation of a window with a Vulkan surface and basic event handling as simple and portable as possible. If you just need to open a simple window with basing event/input handling for your Vulkan projects, this is the tool you need. Library is header-only single file and there is no need for implementation files. Just drop it into you project.

## Simple API
The window API was made so it is as simple as possible. Open a window and create a VkSurface in just two lines:
```c++
#define SURFER_PLATFORM_X11
#include "VulkanSurfer.h"

// Create a window
Surfer::Window * window = Surfer::Window::createWindow("Example window", instance, 800, 600 , 100, 100 );

// Get the surface 
VkSurfaceKHR surface = window->getSurface();

// Draw
while (!window->shouldClose()) {
    // poll for events
    window->pollEvents();
    
    // do your rendering
}

// Don't forget to destroy the window
Surfer::Window::destroyWindow(window);
```

See the full example [here](https://github.com/elliahu/VulkanSurfer/blob/master/example/main.cpp).

## Callback-based event handling
Event handling is done using simple callback system.
```c++
window->registerKeyPressCallback([](Surfer::KeyCode key) {
    std::cout << "Key pressed " << static_cast<unsigned int>(key) << std::endl;
});
```

You can find the source code in the [official repository](https://github.com/elliahu/VulkanSurfer).