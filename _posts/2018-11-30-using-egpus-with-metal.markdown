---
published: true
title: Using eGPUs with Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/egpu.png" alt="book" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learn about using eGPUs with Metal, about switching between integrated/discrete/external GPUs, about telling your app which GPU to use, about choosing the appropriate resource storage mode based on needs, about the bandwidth and which GPU is appropriate for displaying content on screen.  ...</div></div>
layout: post
---

- comparison of Vega 64 to 1080ti and 2080ti
- plug in and show icon on the top bar
- activity monitor 
- system information
- Geekbench for CPU and GPU in both OpenCL/Metal
- check all GPUs in a playground/project

[Use an external graphics processor with your Mac](https://support.apple.com/en-us/HT208544)

[Querying Properties](https://developer.apple.com/documentation/metal/mtldevice#2954889)

[Choosing a Resource Storage Mode in macOS](https://developer.apple.com/documentation/metal/resource_objects/setting_resource_storage_modes/choosing_a_resource_storage_mode_in_macos)

[Device Selection and Fallback for Graphics Rendering](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/device_selection_and_fallback_for_graphics_rendering)

[Device Selection and Fallback for Compute Processing](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/device_selection_and_fallback_for_compute_processing)

[About External GPUs](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/about_external_gpus)

[About Multi-GPU and Multi-Display Setups](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/about_multi-gpu_and_multi-display_setups)

[About GPU Bandwidth](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/about_gpu_bandwidth)

[Handling External GPU Additions and Removals](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/handling_external_gpu_additions_and_removals)

[Getting Different Types of GPUs](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/getting_different_types_of_gpus)

[Getting the GPU that Drives a View's Display](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/getting_the_gpu_that_drives_a_view_s_display)

If you build and run the project you should be able to see something similar: 

![alt text](https://raw.githubusercontent.com/MetalKit/images/master/egpu.png?raw=true "book")

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time! 
