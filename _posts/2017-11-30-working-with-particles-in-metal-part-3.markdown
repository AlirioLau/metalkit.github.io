---
published: true
title: Working with Particles in Metal part 3
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/particles3.png" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Trying a third approach to rendering particles, this time cranking up the number of particles rendered to millions! Using a random generator function to populate the window with particles. Learning about the new dispatchThreads(:) method in Metal 2. Updating positions and velocities on the GPU and drawing "thicker" particles using a basic neighboring formula.</div></div>
layout: post
---
Last time we looked at how to manipulate vertices from `Model I/O` objects on the `GPU`. In this part we are going to show yet another way to create particles using compute threads. We can reuse the playground from last time and we start by modifying the __Particle__ struct in our metal view delegate class to only include two members that we will update on the `GPU` - __position__ and __velocity__:

```swift
struct Particle {
    var position: float2
    var velocity: float2
}
```

We need neither the __timer__ variable, nor the __translate(by:)__ and __update()__ methods anymore so you can delete them. The significant change happens inside the __initializeBuffers()__ method:

```swift
func initializeBuffers() {
    for _ in 0 ..< particleCount {
        let particle = Particle(
        		position: float2(Float(arc4random() %  UInt32(side)), 
        							Float(arc4random() % UInt32(side))), 
        		velocity: float2((Float(arc4random() %  10) - 5) / 10, 
        							(Float(arc4random() %  10) - 5) / 10))
        particles.append(particle)
    }
    let size = particles.count * MemoryLayout<Particle>.size
    particleBuffer = device.makeBuffer(bytes: &particles, length: size, options: [])
}
```

> Note: we generate random positions to fill the entire window and we also generate velocities that will range between `[-5, 5]`. we also divide by `10` to slow them down a little. 

The most impoartant part however, is happening when configuring the command encoder. We set the numbers of `threads per group` to be a `2D` grid determined on one side by the `thread execution width` and on the other side by the `maximum total threads per threadgroup` which are hardware characteristics specific to each `GPU` and will never change during execution. We set the number of `threads per grid` to be a one-dimensional array whose size is determined by the particle count:

```swift
let w = pipelineState.threadExecutionWidth
let h = pipelineState.maxTotalThreadsPerThreadgroup / w
let threadsPerGroup = MTLSizeMake(w, h, 1)
let threadsPerGrid = MTLSizeMake(particleCount, 1, 1)
commandEncoder.dispatchThreads(threadsPerGrid, threadsPerThreadgroup: threadsPerGroup)
```

> Note: new in `Metal 2`, the __dispatchThreads(:)__ method lets us dispatch work without having to specify how many thread groups we want. in contrast to using the older __dispatchThreadgroups(:)__ method, the new method calculates the number of groups and provides `nonuniform thread groups` when the size of the grid is not a multiple of the group size, and also makes sure there are no underutilized threads. 

On to the kernel shader, we first match the particle struct with the one on the `CPU` and then inside the kernel we update the positions and velocities:

```swift
Particle particle = particles[id];
float2 position = particle.position;
float2 velocity = particle.velocity;
int width = output.get_width();
int height = output.get_height();
if (position.x < 0 || position.x > width) { velocity.x *= -1; }
if (position.y < 0 || position.y > height) { velocity.y *= -1; }
position += velocity;
particle.position = position;
particle.velocity = velocity;
particles[id] = particle;
uint2 pos = uint2(position.x, position.y);
output.write(half4(1.), pos);
output.write(half4(1.), pos + uint2( 1, 0));
output.write(half4(1.), pos + uint2( 0, 1));
output.write(half4(1.), pos - uint2( 1, 0));
output.write(half4(1.), pos - uint2( 0, 1));
```
> Note: we do checks for bounds and when that happens we simply reverse the velocity so the particles do not leave the screen. we also use a neat trick when drawing, by making sure the four neighboring particles are also drawn so they look a bit larger.

You can set __particleCount__ to `1,000,000` if you want but it will take a few seconds to generate them before rendering them all. Because I am only rendering in a relatively small window, I am only rendering `10,000` particles so they don't look too crammed in this window space. If you run the app, you should be able to see the particles moving around randomly:

￼￼![alt text](https://github.com/MetalKit/images/blob/master/particles3.gif?raw=true "Particle")

This article concludes the rendering particles series. I want to thank [FlexMonkey](https://twitter.com/flexmonkey) for sharing great insights about compute concepts. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time! 
