---
published: true
title: Metal 2 on the A11 GPU
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/metal2.png" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learn about the new GPU Family 4. Learn about the new Apple-designed GPU inside the A11 Bionic processor. Learn about the new features that Metal 2 brings for the A11-powered devices. Imageblocks, Tile Shading, Raster Order Groups, Imageblock Sample Coverage Control, Threadgroup Sharing. Also brifly look at Face Tracking on A11 using ARKit.</div></div>
layout: post
---
When the new `A11`-powered `iPhone` models (8, 8 Plus and X) were announced in the _September Event keynote_, the new [GPU Family 4](https://developer.apple.com/documentation/metal/about_gpu_family_4) webpage and a series of [new Metal videos](https://developer.apple.com/videos/metal) labeled `Fall 2017` were published. The new processor, called __A11 Bionic__, has the first `Apple-designed GPU` which comes with three cores. Internal tests state it is __30%__ faster than the previous `GPU` inside the `A10`. It also features a new `Neural Engine` hardware addition, for machine learning.

Below is a table I made out of the [Metal Feature Sets](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf) document. It includes only what `Metal 2` introduces for the `A11` devices.

![alt text](https://github.com/MetalKit/images/blob/master/A11.png?raw=true "Particle")

> Note:  I only included features that are new for `A11` and not available for `A10` or earlier. Some of these features are also available for `macOS` devices. 

Let's look briefly into some of these features:

- [Imageblocks](https://developer.apple.com/documentation/metal/about_gpu_family_4/about_imageblocks) - is not a new concept on iOS devices, however, `Metal 2` on `A11` lets us treat imageblocks (which are structured image data in tile memory) as data structures with granular and full control. They are integrated with fragment and compute shaders.  

- [Tile Shading](https://developer.apple.com/documentation/metal/about_gpu_family_4/about_tile_shading) - is a rendering technique that allows fragment and compute shaders to access persistent tile memory between the two rendering phases. Tile memory is GPU on-chip memory that improves performance by storing intermediate results locally instead of using the device memory. Tile memory from one phase is available to any subsequent fragment phases.

- [Raster Order Groups](https://developer.apple.com/documentation/metal/about_gpu_family_4/about_raster_order_groups) - provide ordered memory access from fragment shaders and facilitate features such as order-independent transparency, dual-layer G-buffers, and voxelization.

- [Imageblock Sample Coverage Control](https://developer.apple.com/documentation/metal/about_gpu_family_4/about_enhanced_msaa_and_imageblock_sample_coverage_control) - Metal 2 on A11 tracks the number of unique samples for each pixel, updating this information as new primitives are rendered. The pixel blends one one time less than on A10 or earlier GPUs, when the covered samples share the same color. 

- [Threadgroup Sharing](https://developer.apple.com/documentation/metal/about_gpu_family_4/about_threadgroup_sharing) -  allows threadgroups and the threads within a threadgroup to communicate with each other using atomic operations or a memory fence rather than expensive barriers. 

Even though it is not necessarily `Metal` related, at the same event, the [Face Tracking with ARKit](https://developer.apple.com/videos/play/fall2017/601/) video and [Creating Face-Based AR Experiences](https://developer.apple.com/documentation/arkit/creating_face_based_ar_experiences) webpage were also published. `Face Tracking`, however, is only possible on the `iPhone X` because it is the only one at the moment that has a `TrueDepth` front camera. The most immediate application of face tracking we have all seen during the _September Event keynote_, were the amazing __Animoji__! The new `Neural Engine` hardware is responsible for `FaceID` and `Animoji`, among other machine learning tasks. Since I purchased an `iPhone X` recently, it might give me some ideas for a new post in the `Using ARKit with Metal` series.
 
Until next time! 
