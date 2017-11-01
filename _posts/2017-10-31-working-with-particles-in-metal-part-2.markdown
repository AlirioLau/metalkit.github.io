---
published: true
title: Working with Particles in Metal part 2
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/particles.png" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Trying a different approach to rendering particles, this time moving away from distance fields to using vertex buffers. Cranking up the number of particles rendered to thousands! Using a Model I/O object as a blueprint for the object to be instanced in the vertex shader. Continuously updating the particles positions on the CPU and sending the updates to the GPU.</div></div>
layout: post
---
Last time we looked at how to quickly prototype a particle-like object directly inside a shader, using distance functions. That was acceptable for moving an object based on time elapsed. However, if we want to work with vertices we would need to define the particles on the `CPU` and send the vertex data to the `GPU`. We use again a minimal playground we used in the past for `3D` rendering, and we start by creating a __Particle__ struct in our metal view delegate class:

```swift
struct Particle {
    var initialMatrix = matrix_identity_float4x4
    var matrix = matrix_identity_float4x4
    var color = float4()
}
```

Next, we create an array of particles and a buffer to hold the data. Here we also give each particle a nice blue color and a random position to start at:

```swift
particles = [Particle](repeatElement(Particle(), count: 1000))
particlesBuffer = device.makeBuffer(length: particles.count * MemoryLayout<Particle>.stride, options: [])!
var pointer = particlesBuffer.contents().bindMemory(to: Particle.self, capacity: particles.count)
for _ in particles {
    pointer.pointee.initialMatrix = translate(by: [Float(drand48()) / 10, Float(drand48()) * 10, 0])
    pointer.pointee.color = float4(0.2, 0.6, 0.9, 1)
    pointer = pointer.advanced(by: 1)
}
```

> Note: we divide the `x` coordinate by `10` to gather particles inside a small horizontal range, while we multiply the `y` coordinate by `10` for the opposite effect - to spread out the particles vertically a little. 

The next step is to create a sphere that will serve as the particle's mesh:

```swift
let allocator = MTKMeshBufferAllocator(device: device)
let sphere = MDLMesh(sphereWithExtent: [0.01, 0.01, 0.01], segments: [8, 8], inwardNormals: false, geometryType: .triangles, allocator: allocator)
do { model = try MTKMesh(mesh: sphere, device: device) } 
catch let e { print(e) }
```

Next, we need an updating function to animate the particles on the screen. Inside, we increase the timer each frame by `0.01` and update the `y` coordinate using the timer value - creating a falling-like motion:

```swift
func update() {
    timer += 0.01
    var pointer = particlesBuffer.contents().bindMemory(to: Particle.self, capacity: particles.count)
    for _ in particles {
        pointer.pointee.matrix = translate(by: [0, -3 * timer, 0]) * pointer.pointee.initialMatrix
        pointer = pointer.advanced(by: 1)
    }
}
```

At this point we are ready to call this function inside the __draw__ method and then send the data to the `GPU`:

```swift
update()
let submesh = model.submeshes[0]
commandEncoder.setVertexBuffer(model.vertexBuffers[0].buffer, offset: 0, index: 0)
commandEncoder.setVertexBuffer(particlesBuffer, offset: 0, index: 1)
commandEncoder.drawIndexedPrimitives(type: .triangle, indexCount: submesh.indexCount, indexType: submesh.indexType, indexBuffer: submesh.indexBuffer.buffer, indexBufferOffset: 0, instanceCount: particles.count)
```

In the __Shaders.metal__ file we have a struct for the incoming and outgoing vertices, as well as one for the particle instances:

```clike
struct VertexIn {
    float4 position [[attribute(0)]];
};

struct VertexOut {
    float4 position [[position]];
    float4 color;
};

struct Particle {
    float4x4 initial_matrix;
    float4x4 matrix;
    float4 color;
};
```

The vertex shader uses the __instance_id__ attribute which we use to create many instances of the same one sphere we sent to the `GPU` in the vertex buffer at index `0`. We then assign to each instance one of the positions we stored and sent to the `GPU` in the buffer at index `1`.

```clike
vertex VertexOut vertex_main(const VertexIn vertex_in [[stage_in]],
                             constant Particle *particles [[buffer(1)]],
                             uint instanceid [[instance_id]]) {
    VertexOut vertex_out;
    Particle particle = particles[instanceid];
    vertex_out.position = particle.matrix * vertex_in.position ;
    vertex_out.color = particle.color;
    return vertex_out;
}
```

Finally, in the fragment shader we return the color we passed through in the vertex shader:

```clike
fragment float4 fragment_main(VertexOut vertex_in [[stage_in]]) {
    return vertex_in.color;
}
```

If you run the app, you should be able to see the particles falling down like a water stream:

￼￼![alt text](https://github.com/MetalKit/images/blob/master/particles.gif?raw=true "Particle")

There is yet another, much more efficient approach to rendering particles on the `GPU`. We'll look into that next time. I want to thank [Caroline](https://twitter.com/carolinebegbie) for her valuable assistance with instancing. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time! 
