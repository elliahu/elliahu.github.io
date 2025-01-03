---
draft: true
title: "Volumetric rendering"
description: "Using raymarching to render volumetric data stored as a 3D texture"
date: 2025-01-02T10:00:00+01:00
tags: ["c++", "glsl", "vulkan", "raymarching"]
categories: ["posts", "tutorials"]
author: "Matej Elias"
ShowToc: true
TocOpen: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
cover:
  image: hero.png
  alt: Volumetric data
  relative: false
editPost:
    URL: "https://github.com/elliahu/elliahu.github.io/tree/master/content/posts/volumetric_rendering/"
    Text: "Sugest changes" # edit text
---
## Introduction
In this article, we will take a lok at raymarching algorithm and how to use raymarching to sample a 3D volume stored as 3D texture.