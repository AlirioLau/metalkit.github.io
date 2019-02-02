---
published: true
title: Introduction to compute using Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/egpu.png" alt="book" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learn about Metal compute.</div></div>
layout: post
---

**Compute**, or `GPGPU`, in the world of GPU programming is another approach to programming the GPU besides rendering. Both approaches involve high parallelism on the GPU, the difference being the more granular control you have in compute over how threads are given work to do. This is useful in situations where you would want certain threads to work on one part of the problem and other threads on another part of the same problem.

This post starts a series of articles about compute. The topic in this post is about image processing because it is the easiest way to introduce compute and threads management. 

> Note: this article assumes you know how to create a minimal Metal project or playground that can do as much as clearing the screen to a solid color.

The first different thing you need to do is create a `MTLComputePipelineState` instead of the good ol' `MTLRenderPipelineState` you've been using in the past for rendering. You can create it like this:

```swift
let function = library.makeFunction(name: "compute")
let pipelineState = device.makeComputePipelineState(function: function)
```

A second thing you need is a texture for the threads to work on. If you work in a playground, all you need are these lines:

```swift
let textureLoader = MTKTextureLoader(device: device)
let url = Bundle.main.url(forResource: "nature", withExtension: "jpg")!
let image = try textureLoader.newTexture(URL: url, options: [:])
```

A third thing you need is a `MTLComputeCommandEncoder` object to which you will attach the pipeline state object and the texture you both created earlier: 

```swift
commandEncoder.setComputePipelineState(pipelineState)
commandEncoder.setTexture(image, index: 0)
```

A fourth thing you need is a `kernel shader` which, remember, you created a function called **compute** for it in the very beginning. Of course, you would put your kernel code in the **.metal** file:

```cpp
kernel void compute(texture2d<float, access::read> input [[texture(0)]],
                    texture2d<float, access::write> output [[texture(1)]],
                    uint2 id [[thread_position_in_grid]]) {
    float4 color = input.read(id);
    output.write(color, id);
}
```

In the shader, **input** is the `MTLTexture` object you created earlier and called **image** and **output** is the drawable texture you are writing to, so it can be presented to the screen by the command buffer. 

A fifth and final thing you need is dispatching the threads to do the work. This is where the fun part begins! All you need to do is add these lines before ending the encoding on your `commandEncoder`:

```swift
let threadsPerGroup = MTLSizeMake(100, 10, 1)
let groupsPerGrid = MTLSizeMake(15, 90, 1)
commandEncoder.dispatchThreadgroups(groupsPerGrid, threadsPerThreadgroup: threadsPerGroup)
```

So what's going on there? Threads are being dispatched to process data in grids. Grids can be 1-, 2- or 3-dimensional. In this case you are using a 2D grid because you are processing an image. Regardless of dimensionality, however, grids are always split into multiple groups of threads so this equality will always define a grid:

```
grid = thread_groups * threads_per_group
```

In the case above you defined a group to have `100 x 10` threads and the grid to have `15 x 90` groups. If you run your playground, you should see something like this:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/compute1.png?raw=true "Plugged In")</span>

Yikes, what's with the red bands? That's what could happen to you when you are trying to guess the image size, the number of threads and groups instead of using the "smart" way. :-)

Apparently, the image is larger in both dimensions than the number of threads dispatched. One thing you can do is to use the image size for an educated guess for the number of groups you should use:

```swift
let width = Int(view.drawableSize.width)
let height = Int(view.drawableSize.height)
let w = threadsPerGroup.width
let h = threadsPerGroup.height
let groupsPerGrid = MTLSizeMake(width / w, height / h, 1)
```

Run the playground and the image should look properly now:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/compute2.png?raw=true "Plugged In")</span>

There is another issue that might happen - underutilization. Take a look at this diagram:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/nonuniform.png?raw=true "Plugged In")</span>

> Note: this image belongs to Razeware, the publisher of the Metal by Tutorials book.

Normally, you would think a properly designed grid would be 6 groups of 16 threads (4 x 4) each, so a grid of 12 x 8 threads. Some of the threads at the bottom and right side edges would be underutilized, however, because there is no work for them to do. 

If you made a smaller grid, say 8 x 4, which would only have whole groups, would result in the red bands you already noticed in the beginning. That means the only acceptable solution is to fix the underutilization problem. You can solve this by making sure you are adding an extra group in each dimension, like this:

```swift
let groupsPerGrid = MTLSizeMake((width + w - 1) / w, (height + h - 1) / h, 1) 
```

What you did was to practically enlarge the grid size with an extra `(w, h, 1)`. This now poses another risk - accessing out of bounds coordinates. To deal with this you need to add a check routine to your kernel shader, right before reading from the input image:

```cpp
if (id.x >= output.get_width() || id.y >= output.get_height()) {
    return;
}
```

This will take care of threads that are not supposed to do any work and also that takes care of unexpected accesses. 

What about the size of a thread group - can't that be also optimized? You've been guessing those sizes until now. Of course there is a way to get an optimal size of groups too. The hardware provides some features you can access via the pipeline state object:

```swift
var w = pipelineState.threadExecutionWidth
var h = pipelineState.maxTotalThreadsPerThreadgroup / w
let threadsPerGroup = MTLSizeMake(w, h, 1)
```

That's great! What about finding a way to avoid having these underutilization and bound checks? Metal's got you covered here too. Instead of using `dispatchThreadgroups()` the API provides the newer `dispatchThreads()` function which achieves two great things: 

1. Takes the burden of having to deal with underutilization away from you by auto-creating non-uniform thread groups (eg. `3 x 4`) that will adapt to edge cases.
2. It will even decide how many groups are needed, provided that you give it the grid size and the group size you want to work with.

All you need to do is replace the lines where you calculated the number of groups per grid with this:

```swift
w = Int(view.drawableSize.width)
h = Int(view.drawableSize.height)
let threadsPerGrid = MTLSizeMake(w, h, 1)
commandEncoder.dispatchThreads(threadsPerGrid, threadsPerThreadgroup: threadsPerGroup)
```

But wait, didn't I say that this is the part where the fun happens? All right, then go to the kernel shader and remove the bounds check code since it's now not needed anymore. Then before the last line, add this new line that swaps the color channels around:

```cpp
color = float4(color.g, color.b, color.r, 1.0);
```

Run the playground and the image should look like this:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/compute3.png?raw=true "Plugged In")</span>

Replace that previous line with this line which applies an grayscale to the image:

```cpp
color.xyz = (color.r * 0.3 + color.g * 0.6 + color.b * 0.1) * 1.5;
```

Run the playground and the image should look like this:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/compute5.png?raw=true "Plugged In")</span>

Finally, replace this code:

```cpp
float4 color = input.read(id);
color.xyz = (color.r * 0.3 + color.g * 0.6 + color.b * 0.1) * 1.5;
```

with this code which pixelates the image into 5-px squares (you can play with the sizes):

```cpp
uint2 index = uint2((id.x / 5) * 5, (id.y / 5) * 5);
float4 color = input.read(index);
```

Run the playground and the image should look like this:

<span style="display:block;text-align:center">![alt text](https://raw.githubusercontent.com/MetalKit/images/master/compute4.png?raw=true "Plugged In")</span>
 
Did you have fun? I hope you did, just like I did too. This was just a brief introduction to the amazing power of GPGPU and compute capabilities of your GPU. Stay tuned for new topics.

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time! 
