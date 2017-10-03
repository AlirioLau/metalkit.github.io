---
published: true
title: Working with Particles in Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/particle.png" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Introduction to particle systems. Seeing how a particle is created. Keeping track of its position using a distance function. Using time to emulate gravity. Implementing basic collision detection.</div></div>
layout: post
---
Today, we're going to start a new series about particles in `Metal`. Since most of the time particles are tiny objects, we are not usually concerned about their geometry. This makes them fit for a compute shader because later on we will want to have granular control over particle-particle interactions and this is a case fit for a high degree of parallelism control which a compute shader allows us to have. Let's use the last playground we worked on when we did ambient occlusion and continue from there. That playground is useful here because it already has a __time__ variable that the `CPU` passes to the `GPU`. Let's start with a fresh __Shaders.metal__ file, and just give the background a nice color:

{% highlight swift %}#include <metal_stdlib>
using namespace metal;

kernel void compute(texture2d<float, access::write> output [[texture(0)]],
                    constant float &time [[buffer(0)]],
                    uint2 gid [[thread_position_in_grid]]) {
    float width = output.get_width();
    float height = output.get_height();
    float2 uv = float2(gid) / float2(width, height);
    float aspect = width / height;
    uv.x *= aspect;
    output.write(float4(0.2, 0.5, 0.7, 1), gid);
}
{% endhighlight %}

Next, let's create a particle object that only has a position (center) and a radius:

{% highlight swift %}struct Particle {
    float2 center;
    float radius;
};
{% endhighlight %}

We also need a way to know where the particle is on the screen, so let's create a distance function for that:

{% highlight swift %}float distanceToParticle(float2 point, Particle p) {
    return length(point - p.center) - p.radius;
}
{% endhighlight %}

Inside the kernel, right above the last line, let's create a new particle and place it at the top of the screen, midway on the `X` axis. Give it a radius of `0.05`:

{% highlight swift %}float2 center = float2(aspect / 2, time);
float radius = 0.05;
Particle p = Particle{center, radius};
{% endhighlight %}

> Note: we used the `time` as the `Y` coordinate of the particle but this is only a trick to show basic movement. Soon, we will replace this variable with a coordinate that changes under the laws of physics. 

Replace the last line of the kernel with these lines and run the app. You should see the particle falling down at a steady rate:

{% highlight swift %}float distance = distanceToParticle(uv, p);
float4 color = float4(1, 0.7, 0, 1);
if (distance > 0) { color = float4(0.2, 0.5, 0.7, 1); }
output.write(float4(color), gid);
{% endhighlight %}

The particle, however, will keep going down forever. To make it stop at the bottom, enforce this condition right before creating the particle:

{% highlight swift %}float stop = 1 - radius;
if (time >= stop) { center.y = stop; }
else center.y = time;
{% endhighlight %}

> Note: both `time` and the `uv` variables go from `0-1` so we create a `stop` point which is the window height less the particle radius. 

That was a very basic collision detection rule. If you run the app, you should be able to see the particle falling down uniformly and stopping at the bottom, like this:

￼￼![alt text](https://github.com/MetalKit/images/blob/master/particle.gif?raw=true "Particle")

Next time we will go deeper into particle dynamics and implement the laws of motion from physics. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time! 
