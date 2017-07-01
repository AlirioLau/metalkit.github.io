---
published: true
title: Introducing Metal 2
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/metal2.png" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Read about what's new in Metal, now called Metal 2. MPS has new features but most important - it now works on macOS too. See why the new Argument Buffers are the biggest addition to the framework this year. Read about how Raster Order Groups can help avoiding race conditions in shaders. Find out what the ProMotion and Direct to Display features are, and read about all the other new features introduced this year.</div></div>
layout: post
---
This year's `WWDC` was probably the most important one ever, at least as far as we - the `Metal` developers - are concerned. I can wholeheartedly say it was the [best week of my life](https://twitter.com/gpu3d/status/873049387269738497), for sure!

Let's get to the _Games and Graphics_ news. The `most unexpected` trophy goes to the renaming of `Metal` to __Metal 2__. It has the most significant additions and enhancements since it was first announced in `2014`, true, but let's admit it: no one saw this one coming. The `most anticipated` trophy goes to the new __ARKit__ framework. We are only a few weeks after the keynote and there are already numerous bold and funny _Augmented Reality_ projects out there. [ARKit](https://developer.apple.com/arkit/) integrates with `Metal` easily. Finally, the `most influencing` trophy goes to __VR__. It is because of _Virtual Reality_ that we are now able to achieve lower latency, enhanced framerates, as well as more powerful internal and now also [external GPUs](https://developer.apple.com/development-kit/external-graphics/). 

![alt text](https://github.com/MetalKit/images/blob/master/vr.png?raw=true "VR")

New features were also added to the `Model I/O`, `SpriteKit` and `SceneKit` frameworks. Other interesting additions are the `CoreML` and `Vision` frameworks used for [machine learning](https://developer.apple.com/machine-learning/). This article is only focusing on what's new in `Metal`:

1). __MPS__ - the _Metal Performance Shaders_ are now also available on `macOS` and the new additions to `MPS` include:

* four new image processing primitives (`Image Keypoints`, `Bilinear Rescale`, `Image Statistics`, `Element-wise Arithmetic Operations`).
* new linear algebra objects such as `MPSVector`, `MPSMatrix` and `MPSTemporaryMatrix`, as well as _BLAS-style matrix-matrix and matrix-vector multiplication_ and _LAPACK-style triangular matrix factorization and linear solvers_.
* a dozen new `CNN` primitives.
* the `Binary`, `XNOR`, `Dilated`, `Sub-pixel` and `Transpose` convolutions were added to the already existing `Standard` convolution primitive.
* a new `Neural Network Graph` API was added which is useful for describing neural networks using filter and image nodes.
* the `Recurrent Neural Networks` are now coming to help the `CNNs` one-to-one limitation and implement one-to-many and many-to-many relationships.
        
2). __Argument Buffers__ - likely the most important addition to the framework this year. In the traditional argument model, for each object we would call the various functions to set buffers, textures, samplers linearly and then at the end we would have our draw call for that object.

![alt text](https://github.com/MetalKit/images/blob/master/ArgumentBuffers1.png?raw=true "Argument Buffers 1")

As you can imagine, the number of calls will increase drastically when multiplying the number of calls with the total number of objects and with the number of frames where all these objects need to be drawn. As a consequence this will limit the number of objects that will appear on the screen eventually. 

![alt text](https://github.com/MetalKit/images/blob/master/ArgumentBuffers2.png?raw=true "Argument Buffers 2")

`Argument Buffers` introduce an efficient new way of configuring how to use resources by adopting the _indirect_ behavior that the constants have, and applying it to textures, samplers, states, pointers to other buffers, and so on. The argument buffer will now only have `2 API calls per object`: set the argument buffer and then draw. With this approach many more objects can be drawn. 

![alt text](https://github.com/MetalKit/images/blob/master/ArgumentBuffers3.png?raw=true "Argument Buffers 3")

Using argument buffers is as easy as matching the shader data with the host data:

{% highlight swift %}struct Material {
    float intensity;
    texture2d<float> aTexture;
    sampler aSampler;
}

kernel void compute(constant Material &material [[ buffer(0) ]]) {
    ...
}
{% endhighlight %}

On the `CPU`, the argument buffers are created and used by an __MTLArgumentEncoder__ object and they can be blit between `CPU` and `GPU` easily:

{% highlight swift %}let function = library.makeFunction(name: "compute")
let encoder = function.makeIndirectArgumentEncoder(bufferIndex: 0)
encoder.setTexture(myTexture, index: 0)
encoder.constantData(at: 1).storeBytes(of: myPosition, as: float4)
{% endhighlight %}

But it can get even better using the `dynamic indexing` feature. A great use case is when rendering crowds. An array of argument buffers can pack the data together for all instances (characters). Then, instead of having two calls per object, now we can have only `2 API calls per frame`: one to set the buffer and one to draw indexed primitives for a large instance count! 

![alt text](https://github.com/MetalKit/images/blob/master/ArgumentBuffers4.png?raw=true "Argument Buffers 4")

Then the `GPU` will process per-instance geometry and color. The shader will now take an array of argument buffers as input, dynamically pick the character for any instance index, and return the geometry for that object:

{% highlight swift %}vertex Vertex instanced(constant Character *crowd [[ buffer(0) ]],
                        uint id [[instance_id]]) {
    constant Character &instance = crowd[id];
    ...
}
{% endhighlight %}

Another use case for argument buffers is when running particle simulations. For this we have the `resource setting on the GPU` feature which refers to having an array of argument buffers, one buffer for each particle (thread). All the particle properties (position, material, and so on) are created and stored in argument buffers on the `GPU` so when a particle needs a specific property, such as a material, it will copy it from the argument buffers instead of getting it from the `CPU` thus avoiding expensive copies between them. 

![alt text](https://github.com/MetalKit/images/blob/master/ArgumentBuffers5.png?raw=true "Argument Buffers 5")

A copying kernel is straightforward and lets you assign constant values, do partial or complete copies between a source and a destination object:

{% highlight swift %}kernel void reuse(constant Material &source [[ buffer(0) ]],
                  device Material &destination [[ buffer(1) ]]) {
    destination.intensity = 0.5f;
    destination.aTexture = source.aTexture;
    destination = source;
}
{% endhighlight %}

Finally, we also have the use case of referencing other argument buffers. Imagine a structure to represent an instance (character) that will have a pointer to the `Material` structure such that many instances can point to the same material. Likewise, imagine another structure to represent a tree of nodes where each `Node` would have a pointer to the `Instance` structure which will act as an array of instances in the node:

{% highlight swift %}struct Instance {
    float4 position;
    device Material *material;
}

struct Node {
    device Instance *instances;
}
{% endhighlight %}

> Note: for now, only `Tier 2` devices support all these argument buffer features. Starting with `Metal 2` the `GPU` devices are now classified as either `Tier 1` (integrated) or `Tier 2` (discrete).

3). __Raster Order Groups__ - a new fragment shader synchronization primitive that allows more granular control of the order in which fragment shaders access memory. As an example, when working with custom blending, most graphics `APIs` guarantee that blending happens in draw call order. However, the `GPU` thread parallelism needs a way to prevent race conditions. `Raster Order Groups` do that by providing us with an implicit `Wait` command. 

![alt text](https://github.com/MetalKit/images/blob/master/RasterOrderGroups.png?raw=true "Raster Order Groups")

In traditional blending mode race conditions are created:

{% highlight swift %}fragment void blend(texture2d<float, access::read_write> out[[ texture(0) ]]) {
    float4 newColor = 0.5f;
    // non-atomic memory access without any synchronization
    float4 oldColor = out.read(position);
    float4 blended = someCustomBlendingFunction(newColor, oldColor);
    out.write(blended, position);
}
{% endhighlight %}

All that is needed is adding the `Raster Order Groups` attribute to the texture (or resource) with conflicting accesses:

{% highlight swift %}fragment void blend(texture2d<float, access::read_write> 
				out[[texture(0), raster_order_group(0)]]) {
    float4 newColor = 0.5f;
    // the GPU now waits on first access to raster ordered memory
    float4 oldColor = out.read(position);
    float4 blended = someCustomBlendingFunction(newColor, oldColor);
    out.write(blended, position);
}
{% endhighlight %}

4). __ProMotion__ - only for iPad Pro displays currently. Without `ProMotion` the typical framerate is `60` FPS (`16.6` ms/frame):

![alt text](https://github.com/MetalKit/images/blob/master/promotion1.png?raw=true "ProMotion 1")

With `ProMotion` the framerate goes up to `120` FPS (`8.3` ms/frame) which is really useful for user input such as touch gestures or pencil using:

![alt text](https://github.com/MetalKit/images/blob/master/promotion2.png?raw=true "ProMotion 2")

`ProMotion` also gives us flexibility in when to refresh the display image so we do not need to have a fixed framerate. Without `ProMotion` there is inconsistency in image refreshing which does not cope well with the user experience. Developers usually trade away their peak framerate to constrain all of them to `30` FPS rather than the targeted `48` FPS (`20.83` ms/frame), to achieve consistency:

![alt text](https://github.com/MetalKit/images/blob/master/promotion3.png?raw=true "ProMotion 3")

With `ProMotion` we now have a refresh point every `4` ms rather than every `16` ms (the vertical white lines):

![alt text](https://github.com/MetalKit/images/blob/master/promotion4.png?raw=true "ProMotion 4")

`ProMotion` is also helping in cases of dropped frames. Without `ProMotion` we could have a frame that missed the deadline by taking too long to display:

![alt text](https://github.com/MetalKit/images/blob/master/promotion5.png?raw=true "ProMotion 5")

`ProMotion` fixes this too by only extending the frame with only `4` more ms instead of a whole frame (`16.6` ms):

![alt text](https://github.com/MetalKit/images/blob/master/promotion6.png?raw=true "ProMotion 6")

`UIKit` animations use `ProMotion` automatically but to use `ProMotion` with `Metal` views you need to opt in by disabling the minimum frame duration in the project’s `Info.plist` file. Then you can use one of the __3__ presentation `APIs`. The traditional __present(drawable:)__ will present the image immediately after the `GPU` has finished rendering the frame (`16.6` ms on fixed framerate displays and `4` ms on `ProMotion` displays). The second `API` is __present(drawable, afterMinimumDuration:)__ and provides maximum consistency from frame to frame on fixed framerate displays. The third `API` is __present(drawable, atTime:)__ and is useful when building custom animation loops or when trying to sync the display image with other outputs such as audio. Here is an example of how to implement it:

{% highlight swift %}let targetTime = 0.1
let drawable = metalLayer.nextDrawable()
commandBuffer.present(drawable, atTime: targetTime)
// after 1-2 frames
let presentationDelay = drawable.presentedTime - targetTime
{% endhighlight %}

First, set a time when you want to display the drawable, then render the scene into a command buffer, then wait for the next frame(s) and finally examine the delay so you can adjust the next frame time.

5). __Direct to Display__ - is the new way to send content from the renderer directly to external displays (eg. head mounted devices used in `VR`) with the least amount of latency. There are two paths an image takes after the `GPU` finished rendering it and before it ends on the display. The first one is the typical `UI` scenario when the system is compositing it with other views and layers for a final image:

￼![alt text](https://github.com/MetalKit/images/blob/master/DirectToDisplay1.png?raw=true "Direct To Display 1")

When building a full screen application that does not require blending, scaling or other views/layers, the second path is allowing the display direct access to the memory where we rendered to, thus saving a lot of system resources and avoiding a lot of overhead:

￼￼![alt text](https://github.com/MetalKit/images/blob/master/DirectToDisplay2.png?raw=true "Direct To Display 2")

However, this only happens when certain conditions are met:

* the layer is opaque
* there is no masking or rounded corners
* full screen, or with opaque black bars and background
* the rendered size is at most as large as the display size
* color space and pixel format is compatible with display

The colorspace requirements makes it easier to know when `Direct to Display` mode will work. For example, it is easy to detect if you are using a `P3` display and disable the `P3` mode when trying to use the `Direct to Display` mode.

6). __Other Features__ - include but are not limited to:

* __memory usage queries__ - there are now new `APIs` to query memory use per allocation, as well as total `GPU` memory allocated by the device:
{% highlight swift %}MTLResource.allocatedSize
MTLHeap.currentAllocatedSize
MTLDevice.currentAllocatedSize
{% endhighlight %}
* __SIMDGroup scoped functions__ - allow data sharing between `SIMD` groups directly in the registers by avoiding load/store operations:
￼

￼![alt text](https://github.com/MetalKit/images/blob/master/SIMDGroup.png?raw=true "SIMD Group")

* __non-uniform threadgroup sizes__ - help us not waste `GPU` cycles and avoid working on edge/bound cases:
￼

￼![alt text](https://github.com/MetalKit/images/blob/master/nonuniform.png?raw=true "Non-uniform Threadgroup Sizes")

* __Viewport Arrays__ on `macOS` now support up to `16` viewports for the vertex function to choose from when rendering, and is useful for `VR` when combined with instancing.
* __Multisample Pattern Control__ - allows selecting where within a pixel the `MSAA` sample patters are located and it’s useful for custom anti-aliasing.
* __Resource Heaps__ are now also available on `macOS`. It allows controlling the time of memory allocation, fast reallocation, aliasing of resources and group related resources for faster binding.
* other features include:

|Feature|Description|
|:--|:--|
|`Linear Textures`|Create textures from a `MTLBuffer` without copying.|
|`Function Constant for Argument Indexes`|Specialize bytecodes to change the binding index for shader arguments.|
|`Additional Vertex Array Formats`|Add some 1-/2-component vertex formats and a `BGRA8` vertex format.|
|`IOSurface Textures`|Create `MTLTextures` from `IOSurfaces` on `iOS`.|
|`Dual Source Blending`|Additional blending modes with two source parameters.|
||

I made a table with the most important new features, which states whether the feature is new in the latest version of the operating system or not.
￼

￼![alt text](https://github.com/MetalKit/images/blob/master/features.png?raw=true "Feature Table")

Finally, here are a few lines I wrote to test the differences between my integrated and discrete `GPUs`:
￼

￼![alt text](https://github.com/MetalKit/images/blob/master/gpuCompare.png?raw=true "GPU comparison")

All images were taken from `WWDC` presentations and the [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time!
