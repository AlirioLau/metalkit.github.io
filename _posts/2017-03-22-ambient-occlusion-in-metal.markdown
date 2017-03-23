---
published: true
title: Ambient Occlusion in Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://github.com/MetalKit/images/raw/master/ao_3.png" alt="Metal" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learning about Ambient Occlusion in Metal. Using more distance functions - rectangular shapes. Introducing cone tracing as a derivative of ray tracing using "thicker" rays. Using spheres with different radii to approximate a cone. Marching along the cone instead of a ray, to determine how much light is occluded. Replacing the circling light source with a moving camera so we can examine the entire scene.</div></div>
layout: post
---
Today we will be looking into __ambient occlusion__. We are going to work on the playground we used in [Shadows in Metal part 2](http://metalkit.org/2017/02/28/shadows-in-metal-part-2.html) and build up on that. First, let’s add a new object type - a rectangular box:

{% highlight swift %}struct Box {
    float3 center;
    float size;
    Box(float3 c, float s) {
        center = c;
        size = s;
    }
};
{% endhighlight %}

Next, let’s also add a new distance function for our new struct:

{% highlight swift %}float distToBox(Ray r, Box b) {
    float3 d = abs(r.origin - b.center) - float3(b.size);
    return min(max(d.x, max(d.y, d.z)), 0.0) + length(max(d, 0.0));
}
{% endhighlight %}

Then, update our scene to something new: 

{% highlight swift %}float distToScene(Ray r) {
    Plane p = Plane(0.0);
    float d2p = distToPlane(r, p);
    Sphere s1 = Sphere(float3(0.0, 0.5, 0.0), 8.0);
    Sphere s2 = Sphere(float3(0.0, 0.5, 0.0), 6.0);
    Sphere s3 = Sphere(float3(10., -5., -10.), 15.0);
    Box b = Box(float3(1., 1., -4.), 1.);
    float dtb = distToBox(r, b);
    float d2s1 = distToSphere(r, s1);
    float d2s2 = distToSphere(r, s2);
    float d2s3 = distToSphere(r, s3);
    float dist = differenceOp(d2s1, d2s2);
    dist = differenceOp(dist, d2s3);
    dist = unionOp(dist, dtb);
    dist = unionOp(d2p, dist);
    return dist;
}
{% endhighlight %}

What we did here was to first draw a sphere with a radius of `8`, one with a radius of `6` and take the difference between them. Since they have the same center the smaller one would not be visible unless we made a cross sectioning somehow. That was exactly why we used a third sphere, much larger and with a different center. We took the difference again and we could now see the result of the first difference. Finally, we added a box in there for a nicer, more diverse view. If you run the playground now, you should see something similar: 

![alt text](https://github.com/MetalKit/images/raw/master/ao_1.png "1")

Next, let’s delete the __lighting()__ and __shadow()__ functions as we don’t need them anymore. Also, delete the __Light__ struct and its two instances inside the kernel. Now let's create an `ambient occlusion` surrogate function:

{% highlight swift %}float ao(float3 pos, float3 n) {
    return n.y * 0.5 + 0.5;
}
{% endhighlight %}

We’re just using the normal’s `y` component for light, which is like having a light directly above. Inside the kernel, right after creating the normal (inside the `else` block), call the `ao()` function:

{% highlight swift %}float o = ao(ray.origin, n);
col = col * o;
{% endhighlight %}

There are no shadows anymore, only a basic (directly above) light. If you run the playground now, you should see something similar:

![alt text](https://github.com/MetalKit/images/raw/master/ao_2.png "2")

Time to get some real `ambient occlusion` now. _Ambient_ means the light does not come from a well defined light source but rather means general background lighting. _Occlusion_ means how much ambient light is blocked. We take the point on the surface where our ray hits and look at what’s around it. If there’s an object anywhere around it, that will block most of the light in the scene, so this is a dark area. If there’s nothing around it, then the area is well lit. For in between situations though, we need to figure out more precisely how much light was occluded. Introducing the __cone tracing__ concept.

The idea of `cone tracing` is using a cone in the scene, instead of a ray. If the cone intersects an object, we don’t just have a simple `true/false` result. We can find out how much of the cone the object covers at that point. But how do we even trace a cone? We could make a cone using many spheres. Try to imagine several spheres along a line, very small at one end, big at the other end. This is as good a cone approximation we can get here. Here are the steps we want to take:

- Start at the point on the surface
- March out from the surface, along the normal
- For each iteration, determine how much of the sphere is filled by the scene using distance function
- For each iteration, double the distance from the surface, and also double the size of the sphere

Since we are doubling the sphere size at each step, that means we travel out from the surface very fast so we need fewer iterations. That also gives us a nice wide cone. Here is the complete `ao()` function:

{% highlight swift %}float ao(float3 pos, float3 n) {
    float eps = 0.01;
    pos += n * eps * 2.0;
    float occlusion = 0.0;
    for (float i=1.0; i<10.0; i++) {
        float d = distToScene(Ray(pos, float3(0)));
        float coneWidth = 2.0 * eps;
        float occlusionAmount = max(coneWidth - d, 0.);
        float occlusionFactor = occlusionAmount / coneWidth;
        occlusionFactor *= 1.0 - (i / 10.0);
        occlusion = max(occlusion, occlusionFactor);
        eps *= 2.0;
        pos += n * eps;
    }
    return max(0.0, 1.0 - occlusion);
}
{% endhighlight %}

Let's go over the code, line by line. First we define the __eps__ variable which is both the cone radius and the distance from the surface. Then, we move away a bit to prevent hitting surface we're moving away from. Next, we define the __occlusion__ variable which is initially nil (scene is all lit). Then, we enter the loop and at each iteration we get the scene distance, double the radius so we know how much of the cone is occluded, make sure we eliminate negative values for the light, get the amount (ratio) of occlusion scaled by the cone width, set a lower impact for more distant occluders (the iteration count gives us this), preserve the highest occlusion value so far, double the __eps__ value and finally move along the normal by that distance. We then return a value that represents how much light reaches this point.  

Now lets have a __camera__ struct. It needs a position. Instead of camera direction we'll just store a __ray__. Finally the __rayDivergence__ gives us a factor of how much the ray spreads.

{% highlight swift %}struct Camera {
    float3 position;
    Ray ray = Ray(float3(0), float3(0));
    float rayDivergence;
    Camera(float3 pos, Ray r, float div) {
        position = pos;
        ray = r;
        rayDivergence = div;
    }
};
{% endhighlight %}

Next, we need to set up the camera. It needs the camera position, a look-at target, the field of view and the view coordinates:

{% highlight swift %}Camera setupCam(float3 pos, float3 target, float fov, float2 uv, int x) {
    uv *= fov;
    float3 cw = normalize(target - pos );
    float3 cp = float3(0.0, 1.0, 0.0);
    float3 cu = normalize(cross(cw, cp));
    float3 cv = normalize(cross(cu, cw));
    Ray ray = Ray(pos, normalize(uv.x * cu + uv.y * cv + 0.5 * cw));
    Camera cam = Camera(pos, ray, fov / float(x));
    return cam;
}
{% endhighlight %}

Now we just need to initialize the camera. We'll have it circling the scene, looking at the center __(0,0,0)__. Add this to the kernel, just after you set up the `uv` variable:
 
{% highlight swift %}float3 camPos = float3(sin(time) * 10., 3., cos(time) * 10.);
Camera cam = setupCam(camPos, float3(0), 1.25, uv, width);
{% endhighlight %}
 
Then delete the __ray__ variable, and replace everywhere it was used in the kernel with __cam.ray__ instead. If you run the playground now, you should see something similar:

![alt text](https://github.com/MetalKit/images/raw/master/ao_3.png "3")

To see an animated version of this code, use the `Shadertoy` embedded player below. Just hover over it and click the play button to watch it in action:

<iframe width="740" height="450" frameborder="0" src="https://www.shadertoy.com/embed/4ltSWf" allowfullscreen></iframe><br />

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual. I want to thanks [Chris](https://twitter.com/_psonice) again for his assistance.

Until next time!
