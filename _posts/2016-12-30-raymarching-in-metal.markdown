---
published: true
title: Raymarching in Metal
author: by <a href = "https://twitter.com/MTLDevice" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://github.com/MetalKit/images/raw/master/raymarching.png" alt="Metal" height="150" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learning about Raymarching in Metal. Continue on improving previously used Signed Distance Fields functions. Seeing the similarities and differences from Ray Tracing. Using the ray as camera and marching along with it. Instancing an object to create more complex scenes.</div></div>
layout: post
---
__Raymarching__ is a fast rendering method used in realtime graphics. The geometry is usually not passed to the renderer but rather created in the shader using __Signed Distance Fields (SDF)__ functions that describe the shortest distance between a point and the surface of any object in the scene. The `SDF` returns a negative number if the point is inside of an object. Also, `SDFs` are useful because they allow us to reduce the number of samples used by `Ray Tracing`.  Similar to _Ray Tracing_, in `Raymarching` we also have a ray cast for each pixel on the view plane and each ray is used to determine if there is an intersection with an object. 

The difference between the two techniques is that in _Ray Tracing_ the intersection is determined by a strict set of equations while in `Raymarching` the intersection is approximated. Using `SDFs` we can _march_ along the ray until we get close enough to an object. This is inexpensive to compute compared to exactly determining intersections which could become very expensive when there are many objects in the scene and the lighting is complex. Another great use case for `Raymarching` is volumetric rendering (fog, water, clouds) which _Ray Tracing_ cannot easily do because determining intersections with such volumes is quite difficult.

To follow allong, you can use the playground from [Using MetalKit part 10](http://metalkit.org/2016/05/02/using-metalkit-part-10.html), slightly modified as explained next. Let’s start with two basic building blocks we need at the very minimum in our kernel: a ray and an object (sphere).

{% highlight swift %}struct Ray {
    float3 origin;
    float3 direction;
    Ray(float3 o, float3 d) {
        origin = o;
        direction = d;
    }
};

struct Sphere {
    float3 center;
    float radius;
    Sphere(float3 c, float r) {
        center = c;
        radius = r;
    }
};
{% endhighlight %}

As we did back in _part 10_, let's also write a `SDF` for calculating the distance from a given point to the sphere. The difference from the old function is that now our point is _marching_ along the ray, so we use the ray position instead:

{% highlight swift %}float distToSphere(Ray ray, Sphere s) {
    return length(ray.origin - s.center) - s.radius;
}
{% endhighlight %}

All we had to do back then was to calculate the distance from any given point to a circle (not a sphere because we did not have `3D` back then) like this:

{% highlight swift %}float dist(float2 point, float2 center, float radius) {
    return length(point - center) - radius;
}

...
float distToCircle = dist(uv, float2(0.), 0.5);
bool inside = distToCircle < 0.;
output.write(inside ? float4(1.) : float4(0.), gid);
...
{% endhighlight %}

We now need to have a ray and march along with it through the scene, so replace those last three lines in the kernel with the following lines:

{% highlight swift %}Sphere s = Sphere(float3(0.), 1.);
Ray ray = Ray(float3(0., 0., -3.), normalize(float3(uv, 1.0)));
float3 col = float3(0.);
for (int i=0.; i<100.; i++) {
    float dist = distToSphere(ray, s);
    if (dist < 0.001) {
        col = float3(1.);
        break;
    }
    ray.origin += ray.direction * dist;
}
output.write(float4(col, 1.), gid);
{% endhighlight %}

Let's go over the code line by line. We first create a sphere object and a ray. Notice that as ray's `z` value approaches `0`, the sphere seems bigger because the ray is closer to the scene, and vice versa, when it goes away from `0`, the sphere seems smaller for the obvious reason -- we use our ray as our implicit `camera`. Next we define the color to be initially a solid black. Now comes the very essence of `raymarching`! We loop for a given number of times (steps) to make sure we get enough precision. We go with `100` in this case but you can try with a bigger number of steps and see how the quality of the rendered image improves, at the expense of more computing resources used, however. Inside the loop we calculate the distance from our current position along the ray to the scene, while also checking if we reached an object in the scene, and if we did, we color it in solid white and break out of this particular iteration, or otherwise update the ray position by moving it closer to the scene. 

Notice that we normalized the ray direction to cover edge cases where for example the length of the vector `(1, 1, 1)` (corner of the screen) would have a value of `sqrt(1 * 1 + 1 * 1 + 1 * 1)` which approximates to `1.73`. That means we would need to move the ray position forward by `1.73 * dist` which is almost double the distance we need to move forward, thus making us miss the object by overshooting the ray beyond the intersection point. For that reason, we normalize the direction, to make sure its length will always be `1`. Finally, we write the color to the output texture. If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/raymarching1.png "1")

Now let's create a function named __distToScene__ that only takes a ray in as argument because all we care now is to find the shortest distance to a complex scene that contains multiple objects. Next, we move the code related to the sphere inside this new function where we return only the distance to the sphere (for now). Then, we change the sphere position to `(1, 1, 1)` and its radius to `0.5` which means the sphere is now in the `0.5 ... 1.5` range. Here comes a neat trick to do instancing: if we repeat the space between `0.0 ... 2.0`, the sphere is safely inside. Next, we make a local copy of the ray and modulus its origin. Then we use the repeating ray with the `distToSphere()` function. 

{% highlight swift %}float distToScene(Ray r) {
    Sphere s = Sphere(float3(1.), 0.5);
    Ray repeatRay = r;
    repeatRay.origin = fmod(r.origin, 2.0);
    return distToSphere(repeatRay, s);
}
{% endhighlight %}

By using the `fmod` function we repeated the space throughout the entire screen and pratically created an infinite number of spheres, each with its own (repeated) ray. Of course, we will only see the ones bounded by the `x` and `y` coordinates of the screen, however, the `z` coordinate will let us see how the spheres go indefinitely in depth. Inside the kernel, remove the sphere line, move the ray to a really far location, modify `dist` to rather give us the distance to the scene, and finally change the last line to give us some nice colors:

{% highlight swift %}Ray ray = Ray(float3(1000.), normalize(float3(uv, 1.0)));
...
float dist = distToScene(ray);
...
output.write(float4(col * abs((ray.origin - 1000.) / 10.0), 1.), gid);
{% endhighlight %}

We are multiplying the color by the ray position. We divide by `10.0` because the scene is quite big and the ray position is bigger than `1.0` in most places, which would give us a solid white. We use `abs()` because on the left side of the screen `x` is lower than `0` which would give us a solid black so we basically mirror the top/bottom and left/right colors. Finally, we offset the ray position by `1000` to match the ray origin (camera) we set. If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/raymarching2.png "2")

Next, let's animate the scene! We have seen in [part 12](http://metalkit.org/2016/05/18/using-metalkit-part-12.html) how to send useful _uniforms_ to the `GPU`, such as the `time` so we are not going to discuss about how to implement that again here.

{% highlight swift %}float3 camPos = float3(1000. + sin(time) + 1., 1000. + cos(time) + 1., time);
Ray ray = Ray(camPos, normalize(float3(uv, 1.)));
...
float3 posRelativeToCamera = ray.origin - camPos;
output.write(float4(col * abs((posRelativeToCamera) / 10.0), 1.), gid);
{% endhighlight %}

We added `time` to all three coordinates, but we only fluctuate the `x` and `y` while keeping `z` a straight line. The `1.` part is there just to stop the camera crashing into the nearest sphere. To see an animated version of this code, I used a `Shadertoy` embedded player below. Just hover over it and click the play button to watch it in action:

<iframe width="740" height="450" frameborder="0" src="https://www.shadertoy.com/embed/XtcSDf" allowfullscreen></iframe><br />
    
I want to thanks [Chris](https://twitter.com/_psonice) again for his assistance. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time!
