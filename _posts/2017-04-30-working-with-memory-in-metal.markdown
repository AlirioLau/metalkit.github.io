---
published: true
title: Working with memory in Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://devimages.apple.com.edgekey.net/assets/elements/icons/metal/metal-128x128_2x.png" alt="Metal" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learning about the memory model in Metal. Defining Metal resources as Buffers and Textures. Presenting ways to create a Buffer. Defining the Resource Storage Modes. Presenting the address space qualifiers for function variables and arguments. Looking at UnsafePointer and Pointee as another way to aceess memory locations.</div></div>
layout: post
---
Today we look at how memory is managed when working with the `GPU`. The `Metal` framework defines memory sources as `MTLBuffer` objects which are typeless and unformatted allocations of memory (any type of data), and `MTLTexture` objects which are formatted allocations of memory holding image data. We only look at buffers in this article.

To create `MTLBuffer` objects we have 3 options:

* __makeBuffer(length:options:)__ creates a `MTLBuffer` object with a new allocation.
* __makeBuffer(bytes:length:options:)__ copies data from an existing allocation into a new allocation.
* __makeBuffer(bytesNoCopy:length:options:deallocator:)__ reuses an existing storage allocation.

Let's create a couple of buffers and see how data is being sent to the `GPU` and then sent back to the `CPU`. We first create a buffer for both input and output data and initialize them to some values:

{% highlight swift %}let count = 1500
var myVector = [Float](repeating: 0, count: count)
var length = count * MemoryLayout< Float >.stride
var outBuffer = device.makeBuffer(bytes: myVector, length: length, options: [])
for (index, value) in myVector.enumerated() { myVector[index] = Float(index) }
var inBuffer = device.makeBuffer(bytes: myVector, length: length, options: [])
{% endhighlight %}

The new __MemoryLayout< Type >.stride__ syntax was introduced in `Swift 3` to replace the old `strideof(Type)` function. By the way, we use `.stride` instead of `.size` for memory alignment reasons. The __stride__ is the number of bytes moved when a pointer is incremented. The next step is to tell the command encoder about our buffers:

{% highlight swift %}encoder.setBuffer(inBuffer, offset: 0, at: 0)
encoder.setBuffer(outBuffer, offset: 0, at: 1)
{% endhighlight %}

> Note: the Metal Best Practices Guide states that we should always avoid creating buffers when our data is less than __4 KB__ (up to a thousand `Floats`, for example). In this case we should simply use the __setBytes()__ function instead of creating a buffer. 

The final step is to read the data the `GPU` sent back by using the __contents()__ function to bind the memory data to our output buffer:

{% highlight swift %}let result = outBuffer.contents().bindMemory(to: Float.self, capacity: count)
var data = [Float](repeating:0, count: count)
for i in 0 ..< count { data[i] = result[i] }
{% endhighlight %}

`Metal` resources must be configured for fast memory access and driver performance optimizations. Resource __storage modes__ let us define the storage location and access permissions for our buffers and textures. If you take a look above where we created our buffers, we used the default option (__[]__) as the storage mode. 

All `iOS` and `tvOS` devices support a _unified memory model_ where both the `CPU` and the `GPU` share the system memory, while `macOS` devices support a _discrete memory model_ where the `GPU` has its own memory. In `iOS` and `tvOS`, the __Shared__ mode (`MTLStorageModeShared`) defines system memory accessible to both `CPU` and `GPU`, while __Private__ mode (`MTLStorageModePrivate`) defines system memory accessible only to the GPU. The `Shared` mode is the default storage mode on all three operating systems.

![alt text](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/MTLBestPracticesGuide/Art/ResourceManagement_iOStvOSMemory_2x.png "iOS and tvOS")

Besides these two storage modes, `macOS` also has a __Managed__ mode (`MTLStorageModeManaged`) that defines a synchronized memory pair for a resource, with one copy in system memory and another in video memory for faster CPU and GPU local accesses.
 
![alt text](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/MTLBestPracticesGuide/Art/ResourceManagement_OSXMemory_2x.png "macOS")

Now let's look at what happens on the `GPU` when we send it data buffers. Here is a typical vertex shader example:

{% highlight swift %}vertex Vertices vertex_func(const device Vertices *vertices [[buffer(0)]], 
            		    constant Uniforms &uniforms [[buffer(1)]], 
            		    uint vid [[vertex_id]]) 
{
	...
}
{% endhighlight %}

The `Metal Shading Language` implements address space qualifiers to specify the region of memory where a function variable or argument is allocated: 

* __device__ - refers to buffer memory objects allocated from the device memory pool that are both readable and writeable unless the keyword __const__ preceeds it in which case the objects are only readable.
* __constant__ - refers to buffer memory objects allocated from the device memory pool but that are `read-only`. Variables in program scope must be declared in the constant address space and initialized during the declaration statement. The constant address space is optimized for multiple instances executing a graphics or kernel function accessing the same location in the buffer.
* __threadgroup__ - is used to allocate variables used by a kernel functions only and they are allocated for each threadgroup executing the kernel, are shared by all threads in a threadgroup and exist only for the lifetime of the threadgroup that is executing the kernel. 
* __thread__ - refers to the per-thread memory address space. Variables allocated in this address space are not visible to other threads. Variables declared inside a graphics or kernel function are allocated in the thread address space.

As a bonus, let's also look at another way of accessing memory locations in `Swift 3`. This code snippet belongs to a previous article, [The Model I/O framework](http://metalkit.org/2016/08/30/the-model-i-o-framework.html), so we will not go again into details about voxels. Just think of an array that we need to iterate over to get values:

{% highlight swift %}let url = Bundle.main.url(forResource: "teapot", withExtension: "obj")
let asset = MDLAsset(url: url)
let voxelArray = MDLVoxelArray(asset: asset, divisions: 10, patchRadius: 0)
if let data = voxelArray.voxelIndices() {
    data.withUnsafeBytes { (voxels: UnsafePointer<MDLVoxelIndex>) -> Void in
        let count = data.count / MemoryLayout<MDLVoxelIndex>.size
        let position = voxelArray.spatialLocation(ofIndex: voxels.pointee)
        print(position)
    }
}
{% endhighlight %} 

In this case, the `MDLVoxelArray` object has a function named `spatialLocation()` which lets us iterate through the array by using an `UnsafePointer` of the `MDLVoxelIndex` type and accessing the data through the `pointee` at each location. In this example we are only printing out the first value found at that address but a simple loop will let us get all of them like this:

{% highlight swift %}var voxelIndex = voxels
for _ in 0..<count {
    let position = voxelArray.spatialLocation(ofIndex: voxelIndex.pointee)
    print(position)
    voxelIndex = voxelIndex.successor()
}
{% endhighlight %}

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time!
