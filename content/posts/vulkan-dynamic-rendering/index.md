---
draft: false
title: "Dynamic rendering in Vulkan 1.3"
description: "Exploring dynamic rendering feature of Vulkan 1.3 which aims to simplify rendering by removing need for VkRenderPass and VkFramebuffer objects"
date: 2025-01-02T10:00:00+01:00
tags: ["c++", "vulkan", "computer-graphics"]
categories: ["posts", "tutorials"]
author: "Matej Elias"
ShowToc: true
TocOpen: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
editPost:
    URL: "https://github.com/elliahu/elliahu.github.io/tree/master/content/posts/vulkan-dynamic-rendering/"
    Text: "Sugest changes" # edit text
---
## Introduction
In this article, we explore the `VK_KHR_dynamic_rendering` extension, which eliminates the need for `VkRenderPass` and `VkFramebuffer` objects. By using this extension, rendering attachments (commonly referred to as render targets in other APIs) can be directly referenced before rendering begins.

## How It Works
### Before Dynamic Rendering
Previously, Vulkan required you to create a render pass, specifying the types of attachments to be used. Subpass dependencies could also be defined to handle attachment transitions after the render pass finishes. For example, a subpass could transition a color attachment from `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`, allowing the attachment to be used as input in another pass.

Additionally, a framebuffer object representing the actual attachment images (via their views) had to be created. This framebuffer was statically linked to a specific render pass. For a single logical rendering pass, you needed both a `VkRenderPass` and a `VkFramebuffer`, which had to be properly disposed of when no longer needed. Here’s how this process looked in code:

```cpp
// Use subpass dependencies for attachment layout transitions
std::array<VkSubpassDependency, 2> dependencies{};
// Define the dependencies...

// Create render pass
VkRenderPassCreateInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.pAttachments = attachmentDescriptions.data();
renderPassInfo.attachmentCount = static_cast<uint32_t>(attachmentDescriptions.size());
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;
renderPassInfo.dependencyCount = 2;
renderPassInfo.pDependencies = dependencies.data();
if (vkCreateRenderPass(device.device(), &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("Failed to create render pass");
}

// Create the framebuffer
VkFramebufferCreateInfo framebufferInfo = {};
framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
framebufferInfo.renderPass = renderPass; // Requires render pass
framebufferInfo.pAttachments = attachmentViews.data();
framebufferInfo.attachmentCount = static_cast<uint32_t>(attachmentViews.size());
framebufferInfo.width = width;
framebufferInfo.height = height;
framebufferInfo.layers = maxLayers;
if (vkCreateFramebuffer(device.device(), &framebufferInfo, nullptr, &framebuffer) != VK_SUCCESS) {
    throw std::runtime_error("Failed to create framebuffer");
}

// Begin the render pass
VkRenderPassBeginInfo renderPassBeginInfo = Init::renderPassBeginInfo();
renderPassBeginInfo.renderPass = renderPass;
renderPassBeginInfo.framebuffer = framebuffer;
renderPassBeginInfo.renderArea.extent.width = width;
renderPassBeginInfo.renderArea.extent.height = height;
renderPassBeginInfo.clearValueCount = 3;
renderPassBeginInfo.pClearValues = clearValues.data();

vkCmdBeginRenderPass(drawCmdBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

// Draw the scene

vkCmdEndRenderPass(drawCmdBuffer);
```

#### Note on Graphics Pipelines
When creating a graphics pipeline, the `VkGraphicsPipelineCreateInfo` structure required a valid render pass object. As a result, pipelines were directly tied to specific render passes.

### With Dynamic Rendering
Dynamic rendering eliminates the need for both render pass and framebuffer objects. Instead, attachments are described using the `VkRenderingAttachmentInfoKHR` structure:

```cpp
VkRenderingAttachmentInfoKHR attachmentInfo{};
attachmentInfo.sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO_KHR;
attachmentInfo.imageView = colorAttachment.view;
attachmentInfo.imageLayout = colorAttachment.layout;
attachmentInfo.loadOp = access.loadOp;
attachmentInfo.storeOp = access.storeOp;
attachmentInfo.clearValue = {...};
```

Rendering begins with the `VkRenderingInfoKHR` structure and the `vkCmdBeginRenderingKHR` command:

```cpp
VkRenderingInfoKHR renderingInfo{VK_STRUCTURE_TYPE_RENDERING_INFO_KHR};
renderingInfo.renderArea = {...};
renderingInfo.layerCount = 1;
renderingInfo.colorAttachmentCount = colorAttachments.size();
renderingInfo.pColorAttachments = colorAttachments.data();
renderingInfo.pDepthAttachment = &depthAttachment;
renderingInfo.pStencilAttachment = VK_NULL_HANDLE;

// Start dynamic rendering
vkCmdBeginRenderingKHR(renderContext.getCurrentCommandBuffer(), &renderingInfo);

// Draw the scene

// End rendering
vkCmdEndRenderingKHR(renderContext.getCurrentCommandBuffer());
```

Looks simpler? It is.

#### Graphics Pipelines
Previously, pipeline objects required a pointer to a valid render pass. With dynamic rendering, you can simply set the `renderPass` field in `VkGraphicsPipelineCreateInfo` to `VK_NULL_HANDLE`. To support dynamic rendering, attach a `VkPipelineRenderingCreateInfoKHR` structure to the `pNext` field of `VkGraphicsPipelineCreateInfo`:

```cpp
VkPipelineRenderingCreateInfoKHR pipelineCreate{VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO_KHR};
pipelineCreate.pNext = VK_NULL_HANDLE;
pipelineCreate.colorAttachmentCount = colorAttachmentFormats.size();
pipelineCreate.pColorAttachmentFormats = colorAttachmentFormats.data();
pipelineCreate.depthAttachmentFormat = depthFormat;
pipelineCreate.stencilAttachmentFormat = depthFormat;

// Use pNext to reference the pipelineCreate structure
VkGraphicsPipelineCreateInfo graphicsCreate{VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO};
graphicsCreate.pNext = &pipelineCreate;
graphicsCreate.renderPass = VK_NULL_HANDLE; // Now optional
```

And that’s it.

## Benefits
Why use dynamic rendering? Here are some key advantages:

1. **Simplified API**: Render passes add hidden complexity by automatically transitioning images, which can be hard to trace. Dynamic rendering exposes transitions directly, offering more control to developers.

2. **Ease of Render Graph Implementation**: Building a per-frame render graph (or frame graph) is challenging with static render passes and framebuffers. Dynamic rendering eliminates the need for object pooling and pass matching, reducing the workload.

3. **Improved Resource Management**: Managing `VkFramebuffer` and `VkRenderPass` objects is cumbersome, especially when render target lifespans are unpredictable. Dynamic rendering simplifies this process and reduces CPU overhead.

4. **Flexibility in Pipeline Design**: With dynamic rendering, pipelines are no longer tied to specific render passes. This decoupling allows greater flexibility when designing and reusing pipelines across different rendering setups.

5. **Reduced Boilerplate Code**: By removing the need for `VkRenderPass` and `VkFramebuffer` objects, dynamic rendering significantly reduces the amount of boilerplate code, making Vulkan applications easier to write and maintain.

6. **Simplified Attachment Management**: Managing attachments is more intuitive with dynamic rendering. You can directly specify the attachments during rendering without needing to predefine them in a render pass.

## Performance Considerations
On desktop GPUs, performance differences between dynamic rendering and traditional render passes are negligible. While desktop GPUs can occasionally [benefit from additional information](https://gpuopen.com/learn/vulkan-renderpasses/) provided by render passes, this is mostly relevant for mobile GPUs, where the driver optimizations are more pronounced.

Dynamic rendering is ideal when you don’t need the specific advantages of traditional render passes, providing a low-overhead, streamlined API for most applications.

## Additional Reading
- [Vulkan Do's and Don'ts](https://developer.nvidia.com/blog/vulkan-dos-donts/)
- [RDNA Performance Guide](https://gpuopen.com/learn/rdna-performance-guide/)

