---
published: true
title: Using eGPUs with Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/egpu.png" alt="book" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learn about using eGPUs with Metal, about switching between integrated, discrete or external GPUs, about telling your app which GPU to use, about choosing the appropriate resource storage mode based on needs, about the bandwidth and which GPU is appropriate for displaying content on screen.</div></div>
layout: post
---

For those (like me) who need raw GPU power but only possess a laptop and do not want to buy a beefy desktop machine, the solution seems to be an external GPU (eGPU). While Nvidia eGPUs are not supported by macOS at the moment, there are plenty of GPU options from AMD. 

In looking for the perfect GPU, I stopped at none other than the [AMD Radeon RX Vega 64](https://www.techpowerup.com/gpu-specs/radeon-rx-vega-64.c2871) which is the high end GPU they offer currently and the 2nd most performant among all GPUs on the consumer market. It is topped only by one Nvidia GPU in terms of TFLOPS performance.

A teraflops (TFLOPS) chip is able to run one trillion floating-point operations per second. TFLOPS is a key performance metric in scientific computing, machine learning and any other areas that need compute-intensive work. 

The AMD Radeon RX Vega 64 has 4096 cores able to provide 12 TFLOPS (single precision) and sits conveniently right in between the [Nvidia Geforce GTX 1080 ti](https://www.techpowerup.com/gpu-specs/geforce-gtx-1080-ti.c2877) with 3584 cores (11 TFLOPS) and the new [Nvidia Geforce RTX 2080 ti](https://www.techpowerup.com/gpu-specs/geforce-rtx-2080-ti.c3305) with 4352 cores (13 TFLOPS).

Of course, eGPUs are also accelerating graphics applications and games, lets you connect additional monitors and VR headsets. Keep in mind that eGPUs only work with Thunderbolt 3-equipped Macs running macOS High Sierra 10.13.4 or later. For more informations about compatibility read the [Use an external graphics processor with your Mac](https://support.apple.com/en-us/HT208544) webpage.

I went ahead and purchased a Razer Core X enclosure because for Vega 64 it is recommended to have a power source of at least 600W and there aren't many such boxes available. The [Sonnet eGFX Breakaway box 650](https://www.sonnetstore.com/collections/egpu-expansion-systems/products/egfx-breakaway-box-650) is lighter but more expensive. Razer Core X is as wide and tall as a 15" Macbook Pro as you can see below:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/razer.jpg?raw=true "Razer")</span>

Those 650W are not used entirely by the GPU, by the way. Vega 64 requires only 295W actually. The additional power is available for cases when you overclock your GPU so that it runs even faster, but not only for that - you get to also charge your Mac via the Thunderbolt 3 cable so you do not need the Mac charger anymore while connected to the eGPU. Also, make no mistake, the Vega 64 is almost as wide and heavy as the enclosure - a real monster!

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/vega64.png?raw=true "Vega 64")</span>

As soon as you connect the eGPU to your Mac via a Thunderbolt 3 cable and power up the enclosure, you will notice the new eGPU icon in the menu bar:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/plugged_in.png?raw=true "Plugged In")</span>

In the Activity Monitor if you open the GPU History view you will see all your GPUs listed - integrated, discrete or external:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/activity_monitor.png?raw=true "Activity Monitor")</span>

In the System Information app, under Graphics/Displays, you will see all your GPUs listed as well, along with some basic information:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/system_information.png?raw=true "System Information")</span>

If you right click on a game or application that needs a GPU and click Get Info, you will notice the "Prefer external GPU" option:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/prefer.png?raw=true "Prefer external GPU")</span>

I installed Geekbench 4 so I can run a few benchmarks. The trial version lets you run benchmarks and store results online only and it only lets you run the OpenCL tests. The full version allows for Dropbox integration, saving the results locally and running Metal tests as well.

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/geekbench.png?raw=true "Geekbench")</span>

Running a test only takes a minute to complete:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/geekbench_metal.png?raw=true "Geekbench Metal")</span>

As expected, the biggest score was obtained for a Metal test running on Vega 64. I had the following scores in decreasing order of scores:

-- Metal on Radeon RX Vega 64 -- 137651 <br />
-- OpenCL on Radeon RX Vega 64 -- 135711 <br />
-- Metal on Radeon Pro 450 - 41602 <br />
-- OpenCL on Radeon Pro 450 - 41578 <br />
-- Metal on Intel HD 530 -- 21888 <br />
-- OpenCL on Intel HD 530 -- 20878 <br />
-- OpenCL on quad-core CPU - 13867

The next obvious step is to run some Metal code on these GPUs. Finally! In a playground add this code snippet:

```swift
import Metal

let devices = MTLCopyAllDevices()

for device in devices {
    print(device.name)
    print("Is device low power? \(device.isLowPower).")
    print("Is device external? \(device.isRemovable).")
    print("Maximum threads per group: \(device.maxThreadsPerThreadgroup).")
    print("Maximum buffer length: \(Float(device.maxBufferLength) / 1024 / 1024 / 1024) GB.")
}
```

Run the playground and see a similar output:

```
AMD Radeon RX Vega 64
Is device low power? false.
Is device external? true.
Maximum threads per group: MTLSize(width: 1024, height: 1024, depth: 1024).
Maximum buffer length: 4.5 GB.

AMD Radeon Pro 450
Is device low power? false.
Is device external? false.
Maximum threads per group: MTLSize(width: 1024, height: 1024, depth: 1024).
Maximum buffer length: 1.5 GB.

Intel(R) HD Graphics 530
Is device low power? true.
Is device external? false.
Maximum threads per group: MTLSize(width: 256, height: 256, depth: 256).
Maximum buffer length: 2.0 GB.
```

You can query your devices for many more attributes and features such as memory availability, programmable sample positions support, raster order groups support and so on. For more information, see the [MTLDevice](https://developer.apple.com/documentation/metal/mtldevice) webpage.

Apple provides two sample code projects to help you with GPU management in both rendering and compute pipelines:

- [Device Selection and Fallback for Graphics Rendering](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/device_selection_and_fallback_for_graphics_rendering)
- [Device Selection and Fallback for Compute Processing](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/device_selection_and_fallback_for_compute_processing)

There are also a few webpages with useful information about resource storage modes, about managing multiple displays and GPUs, about GPU bandwidth, about adding/removing external GPUs, and so on:

- [Choosing a Resource Storage Mode in macOS](https://developer.apple.com/documentation/metal/resource_objects/setting_resource_storage_modes/choosing_a_resource_storage_mode_in_macos)
- [About Multi-GPU and Multi-Display Setups](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/about_multi-gpu_and_multi-display_setups)
- [About GPU Bandwidth](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/about_gpu_bandwidth)
- [Handling External GPU Additions and Removals](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/handling_external_gpu_additions_and_removals)
- [Getting Different Types of GPUs](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/getting_different_types_of_gpus)
- [Getting the GPU that Drives a View's Display](https://developer.apple.com/documentation/metal/choosing_gpus_on_mac/getting_the_gpu_that_drives_a_view_s_display)

Until next time! 
