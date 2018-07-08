---
published: true
title: Shadows in Metal part 1
author: by <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://github.com/MetalKit/images/raw/master/shadows.png" alt="Metal" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learning about Shadows in Metal. Building custom functions and operators such as modulus. Creating a light source and animating it around the scene. Creating a shadows function based on the distance and direction to the light. Marching along the light ray and improving the number of steps needed for faster shadows.</div></div>
layout: post
---
A quite important topic in `Computer Graphics` is _lighting and shadows_. This will be a first episode from a multi-part series about __Shadows__ in `Metal`. We are going to work on the playground we used in [Using metal part 15](http://metalkit.org/2016/06/23/using-metalkit-part-15.html) and build up on that. Let’s set up a basic scene: 

{% highlight swift %}float differenceOp(float d0, float d1) {
    return max(d0, -d1);
}

float distanceToRect( float2 point, float2 center, float2 size ) {
    point -= center;
    point = abs(point);
    point -= size / 2.;
    return max(point.x, point.y);
}

float distanceToScene( float2 point ) {
    float d2r1 = distanceToRect( point, float2(0.), float2(0.45, 0.85) );
    float2 mod = point - 0.1 * floor(point / 0.1);
    float d2r2 = distanceToRect( mod, float2( 0.05 ), float2(0.02, 0.04) );
    float diff = differenceOp(d2r1, d2r2);
    return diff;
}
{% endhighlight %}

We first created the __differenceOp()__ function which returns the difference between two signed distances. This comes in handy when we want to _carve_ shapes out of other shapes. Next, we created the __distanceToRect()__ function which determines if a given point is either inside or outside a rectangle. On the `1st` line we offset the current coordinates by the given center. On the `2nd` line we get the symmetrical coordinates of the given point. On the `3rd` line we get the distance to any edge. Then, we created the __distanceToScene()__ function which gives us the closest distance to any object in the scene. Note that the `fmod()` function in `MSL` uses `trunc()` instead of `floor()` so we need to create a custom __mod__ operator here because we also want to use the negative values, so we use the `GLSL` specification for `mod()` which is `x - y * floor(x/y)`. We need the `modulus` operator to draw many small rectangles mirrored on a distance of __0.1__ from each other. Finally, we use these functions to generate a shape that looks a bit like a tall building with windows:

{% highlight swift %}kernel void compute(texture2d<float, access::write> output [[texture(0)]],
                    constant float &timer [[buffer(0)]],
                    uint2 gid [[thread_position_in_grid]])
{
    int width = output.get_width();
    int height = output.get_height();
    float2 uv = float2(gid) / float2(width, height);
    uv = uv * 2.0 - 1.0;
    float d2scene = distanceToScene(uv);
    bool i = d2scene < 0.0;
    float4 color = i ? float4( .1, .5, .5, 1. ) : float4( .7, .8, .8, 1. );
    output.write(color, gid);
}
{% endhighlight %}

If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_1.png "1")

For shadows to work we need to first - get the distance to the light, second - get the direction to the light, and third - step in that direction until we either reach the light or hit an object. So let's create a light at position __lightPos__ which we will animate for fun. We use that good old __timer__ uniform that we have it handy passed from the host (`API`) code. Then, we get the distance from any given point to `lightPos` and then just color the pixel based on the distance from the light - if not inside an object. We want the color to be lighter closer to the light and darker when further away. We use the `max()` function to avoid negative values for the brightness of the light. Replace the last line in the kernel with the lines below:
  
{% highlight swift %}float2 lightPos = float2(1.3 * sin(timer), 1.3 * cos(timer));
float dist2light = length(lightPos - uv);
color *= max(0.0, 2. - dist2light );
output.write(color, gid);
{% endhighlight %}  

If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_2.png "2")

We did the first two steps (light position and direction) so let's proceed to doing the third one - the actual shadow function:

{% highlight swift %}float getShadow(float2 point, float2 lightPos) {
    float2 lightDir = lightPos - point;
    float dist2light = length(lightDir);
    for (float i=0.; i < 300.; i++) {
        float distAlongRay = dist2light * (i / 300.);
        float2 currentPoint = point + lightDir * distAlongRay;
        float d2scene = distanceToScene(currentPoint);
        if (d2scene <= 0.) { return 0.; }
    }
    return 1.;
} 
{% endhighlight %}

Let's go over the code, line by line. We first get the direction from the point to the light. Next, we find the distance to the light so we know how far we need to move along this light ray. Then, we use a loop to divide the ray into many smaller steps. If we don't use enough steps, we might jump past our object and that would leave "holes" in the shadow. Next, we calculate how far along the ray we are currently and move along the ray by this distance to find the point in space we're sampling at. Then, we see how far we are from the surface at that point and then test if we are inside an object. If we are, return __0__ because we are in the shadow, otherwise return __1__ as the ray did not hit any object. It is finally time to see some shadows! In the kernel, replace the last line with the lines below:

{% highlight swift %}float shadow = getShadow(uv, lightPos);
shadow = shadow * 0.5 + 0.5;
color *= shadow;
output.write(color, gid);
{% endhighlight %}

We use the value __0.5__ to attenuate the effect of the shadow, however, feel free to play with various values and notice how it affects itc. If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_3.png "3")

Right now the loop goes in one-pixel steps which is not good performance-wise. We can improve that a little by accelerating the steps along the ray. We don't need to move in really small steps. We can move in big steps so long as we don't step past our object. We can safely step in _any_ direction by the distance to the scene instead of a fixed step size, and this way we skip over empty areas really fast! When finding the distance to the nearest surface, we don't know what direction the surface is in so in fact we have the _radius_ of a circle that intersects with the nearest part of the scene. We can trace along the ray, always stepping to the edge of the circle, until the circle radius becomes __0__ which means it intersected a surface. Oh, right, this is the __raymarching__ technique we learned about last time! Simply replace the content of the __getShadow()__ function with the lines below:

{% highlight swift %}float2 lightDir = normalize(lightPos - point);
float dist2light = length(lightDir);
float distAlongRay = 0.0;
for (float i=0.0; i < 80.; i++) {
    float2 currentPoint = point + lightDir * distAlongRay;
    float d2scene = distanceToScene(currentPoint);
    if (d2scene <= 0.001) { return 0.0; }
    distAlongRay += d2scene;
    if (distAlongRay > dist2light) { break; }
}
return 1.;
{% endhighlight %}
 
In `raymarching` the size of the step depends on the distance from the surface. In empty areas it jumps big distances and it can travel a long way. But if it’s parallel to the object and close to it, the distance is always small so the jump size is also small. That means the ray travels very slowly. With a fixed number of steps, it doesn’t travel far. With __80__ or more steps we should be safe from getting "holes" in the shadow. If you run the playground again the output looks similar except the shadow is faster now. To see an animated version of this code, use the `Shadertoy` embedded player below. Just hover over it and click the play button to watch it in action:

<iframe width="740" height="450" frameborder="0" src="https://www.shadertoy.com/embed/lt3SzB" allowfullscreen></iframe><br />

This type of shadows is called `hard shadows`. Next time we will be looking into `soft shadows` which are more realistic and better looking. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time!
