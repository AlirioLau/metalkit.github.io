---
published: true
title: Working with memory in Metal part 2
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/metal.png" alt="Metal" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Looking again at the memory models in Metal. Analyzing data copy vs no-copy methods. Understanding stack vs heap allocations. Page aligning the data using posix_memalign(). Looking again at the resource storage modes - Shared, Private, Memoryless and Managed. Considerations about resource coherency and synchronization between CPU and GPU.</div></div>
layout: post
---
There are a couple of topics we need to discuss in more depth about working with memory. Last time we have seen that to create `MTLBuffer` objects we have 3 options: by creating a new memory allocation with new data, by copying data from an existing allocation into a new allocation and by reusing an existing storage allocation which does not copy data. Since we haven't looked at the memory before, let's check that is actually true. First we copy data into another allocation:
 
{% highlight swift %}let count = 2000
let length = count * MemoryLayout< Float >.stride
var myVector = [Float](repeating: 0, count: count)
let myBuffer = device.makeBuffer(bytes: myVector, length: length, options: [])
withUnsafePointer(to: &myVector) { print($0) }
print(myBuffer.contents())
{% endhighlight %}
 
> Note: the __withUnsafePointer()__ function gives us the memory address of the actual data on the heap instead of the address of the pointer (from the stack) that wraps that data.
 
Your output should look similar to this:
 
{% highlight swift %}0x000000010043e0e0
0x0000000102afd000
{% endhighlight %}

The two data buffers are definitely stored at different memory locations. Now let's use the `no-copy` option:
 
{% highlight swift %}var memory: UnsafeMutableRawPointer? = nil
let alignment = 0x1000
let allocationSize = (length + alignment - 1) & (~(alignment - 1))
posix_memalign(&memory, alignment, allocationSize)
let myBuffer = device.makeBuffer(bytesNoCopy: memory!, 
				 length: allocationSize, 
				 options: [], 
				 deallocator: { (pointer: UnsafeMutableRawPointer, _: Int) in 
					free(pointer) 
				 })
print(memory!)
print(myBuffer.contents())
{% endhighlight %}
 
First, we create a pointer to our data which (data) we'll store on the heap. For this we need to page-align it. We set the page size to be `4K` (`1000` in hexadecimal). We need to also round the buffer size to match the alignment. We used a bitwise `AND` to avoid division which is a very expensive operation. Otherwise we would just round like this: 
 
{% highlight swift %}let allocationSize = ((length + alignment - 1) / alignment) * alignment
{% endhighlight %}
 
Your output should look similar to this:
 
{% highlight swift %}0x000000010300c000
0x000000010300c000
{% endhighlight %}
 
Notice the last three digits in the addresses above? Those come from page-alinging the data because an address is determined by `0 mod pageSize`, hence the last three `0`'s, which makes sense since our page size is `0x1000`.
 
Let's now move to `Storage Modes` which we briefly mentioned last time. There are basically only four main rules to keep in mind, one for each of the storage modes:
 
|Mode|Description|
|:--|:--|
|`Shared`|_default_ on `macOS` buffers, `iOS/tvOS` resources; not available on `macOS` textures.|
|`Private`|mostly use when data is only accessed by `GPU`.|
|`Memoryless`|only for `iOS/tvOS` on-chip temporary render targets (textures).|
|`Managed`|_default_ mode for `macOS` textures; not available on `iOS/tvOS` resources.|
||
 
For a better big picture, here is the full cheat sheet in case you might find it easier to use than remembering the rules above:
 
![alt text](https://github.com/MetalKit/images/blob/master/storage-modes.png?raw=true "Storage Modes")
 
The most complicated case is when working with `macOS` buffers and when the data needs to be accessed by both the `CPU` and the `GPU`. We choose the storage mode based on whether one or more of the following conditions are true:
 
* __Private__ - for large-sized data that changes at most once, so it is not "dirty" at all. create a source buffer with a `Shared` mode and then blit its data into a destination buffer with a `Private` mode. resource coherency is not necessary in this case as the data is only accessed by the `GPU`. this operation is the least expensive (a one-time cost).
* __Managed__ - for medium-sized data that changes infrequently (every few frames), so it is partially "dirty". one copy of the data is stored in system memory for the `CPU` and another copy is stored in `GPU` memory. resource coherency is explicitly managed by synchronizing the two copies.
* __Shared__ - for small-sized data that is updated every frame, so it is fully dirty. data resides in the system memory and is visible and modifyable by both the `CPU` and the `GPU`. resource coherency is only guaranteed within command buffer boundaries.
 
How to make sure coherency is guaranteed? First, make sure that all the modifications done by the `CPU` finished before the command buffer was committed (check if the command buffer status property is __MTLCommandBufferStatusCommitted__). After the `GPU` finishes executing the command buffer, the `CPU` should only start making modifications again only after the `GPU` is signaling the `CPU` that the command buffer finished executing (check if the command buffer status property is __MTLCommandBufferStatusCompleted__).
 
Finally, let's see how synchronization is done for `macOS` resources. For buffers: after a `CPU` write use __didModifyRange()__ to inform the `GPU` of the changes so `Metal` can update that data region only; after a `GPU` write use __synchronize(resource:)__ within a blit operation, to refresh the caches so the `CPU` can access the updated data. For textures: after a `CPU` write use one of the two __replace()__ region functions to inform the `GPU` of the changes so `Metal` can update that data region only; after a `GPU` write use one of the two __synchronize()__ functions within a blit operation to allow `Metal` to update the system memory copy after the `GPU` finished modifying the data. 
 
The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time!
