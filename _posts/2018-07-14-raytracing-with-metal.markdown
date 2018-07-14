---
published: true
title: Raytracing with Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/raytracing-mps.png" alt="book" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learn about the newest additions to the Metal Performance Shaders framework for accelerating ray tracing.</div></div>
layout: post
---

This is going to be a really short post because of two reasons:

1. This concept is explained by `Apple` really well in [this documentation page](https://developer.apple.com/documentation/metalperformanceshaders/metal_for_accelerating_ray_tracing). All I did was to port their code from `Objective-C` to `Swift` since I have not seen a `Swift` version to date.
2. My book, [Metal by Tutorials](https://store.raywenderlich.com/products/metal-by-tutorials), will have an entire chapter dedicated to the `Metal Performance Shaders` framework and raytracing.

In a nutshell, the `Metal Performance Shaders` framework now has a high performance intersector that helps speeding up ray-triangle intersections using an acceleration structure that contains all the vertices in the scene necessary for calculating intersections.

This project is only capable of rendering planes and boxes but in [Metal by Tutorials](https://store.raywenderlich.com/products/metal-by-tutorials) you will be learning about how to render any shape or volume because you will be using a model loader.

If you build and run the project you should be able to see something similar: 

![alt text](https://raw.githubusercontent.com/MetalKit/images/master/raytracing-mps.png?raw=true "book")

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time! 
