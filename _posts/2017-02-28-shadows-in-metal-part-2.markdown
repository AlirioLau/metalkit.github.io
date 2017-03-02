---
published: true
title: Shadows in Metal part 2
author: by <a href = "https://twitter.com/MTLDevice" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://github.com/MetalKit/images/raw/master/shadows_7.png" alt="Metal" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Learning about Soft Shadows in Metal. Returning to Raymarching to build a complex 3D scene using distance operation functions. Learning how to calculate normals and the visualizing them. Learning how to use normals to calculate diffuse and specular lighting. Determining hard shadows as well as soft shadows for objects in a 3D scene.</div></div>
layout: post
---
In this second part of the series, we will be looking into __soft shadows__. We are going to work on the playground we used in [Raymarching in Metal](http://metalkit.org/2016/12/30/raymarching-in-metal.html) and build up on that because it was already set up for `3D` objects. Let’s set up a basic scene that has a sphere, a plane, a light and a ray: 

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

struct Plane {
    float yCoord;
    Plane(float y) {
        yCoord = y;
    }
};

struct Light {
    float3 position;
    Light(float3 pos) {
        position = pos;
    }
};
{% endhighlight %}

Next, we create a few `distance operation` functions that help us determine distances between elements of the scene: 

{% highlight swift %}float unionOp(float d0, float d1) {
    return min(d0, d1);
}

float differenceOp(float d0, float d1) {
    return max(d0, -d1);
}

float distToSphere(Ray ray, Sphere s) {
    return length(ray.origin - s.center) - s.radius;
}

float distToPlane(Ray ray, Plane plane) {
    return ray.origin.y - plane.yCoord;
}
{% endhighlight %}

Next, we create a __distanceToScene()__ function which gives us the closest distance to any object in the scene. We use these functions to generate a shape that looks like a hollow sphere with holes:

{% highlight swift %}float distToScene(Ray r) {
    Plane p = Plane(0.0);
    float d2p = distToPlane(r, p);
    Sphere s1 = Sphere(float3(2.0), 1.9);
    Sphere s2 = Sphere(float3(0.0, 4.0, 0.0), 4.0);
    Sphere s3 = Sphere(float3(0.0, 4.0, 0.0), 3.9);
    Ray repeatRay = r;
    repeatRay.origin = fract(r.origin / 4.0) * 4.0;
    float d2s1 = distToSphere(repeatRay, s1);
    float d2s2 = distToSphere(r, s2);
    float d2s3 = distToSphere(r, s3);
    float dist = differenceOp(d2s2, d2s3);
    dist = differenceOp(dist, d2s1);
    dist = unionOp(d2p, dist);
    return dist;
}
{% endhighlight %}

Everything we wrote so far is old code, just refactored from the _Raymarching_ article. Let's talk about __normals__ and why they are needed. If we have a flat floor - like our plane - the normal is always `(0, 1, 0)`, that is, pointing up. This case is trivial though. The normal in `3D` space is a `float3` and we need to know its position on the ray. Assume the ray just touches the left side of the sphere. The normal should be `(-1, 0, 0)`, that is, pointing to the left and away from the sphere. If the ray moves slightly to the right of that point, it’s inside the sphere `(eg. -0.001)`. If the ray moves slightly to the left, it’s outside the sphere `(eg. 0.001)`. If we subtract left from right we get `-0.001 - 0.001 = -0.002` which points to the left, so this is our `x` coordinate of the normal. We then repeat this for `y` and `z`. We use a `2D` vector named __eps__ so we can easily do [vector swizzling](https://en.wikipedia.org/wiki/Swizzling_(computer_graphics)) using the chosen value `0.001` for various coordinates as needed in each case: 

{% highlight swift %}float3 getNormal(Ray ray) {
    float2 eps = float2(0.001, 0.0);
    float3 n = float3(distToScene(Ray(ray.origin + eps.xyy, ray.direction)) -
                      distToScene(Ray(ray.origin - eps.xyy, ray.direction)),
                      distToScene(Ray(ray.origin + eps.yxy, ray.direction)) -
                      distToScene(Ray(ray.origin - eps.yxy, ray.direction)),
                      distToScene(Ray(ray.origin + eps.yyx, ray.direction)) -
                      distToScene(Ray(ray.origin - eps.yyx, ray.direction)));
    return normalize(n);
}
{% endhighlight %}

Finally, we are ready to see some visuals. We again use the old `Raymarching` code and at the end of the kernel function we just add the normal so we can interpolate it with the color for every pixel:

{% highlight swift %}kernel void compute(texture2d<float, access::write> output [[texture(0)]],
                    constant float &time [[buffer(0)]],
                    uint2 gid [[thread_position_in_grid]]) {
    int width = output.get_width();
    int height = output.get_height();
    float2 uv = float2(gid) / float2(width, height);
    uv = uv * 2.0 - 1.0;
    uv.y = -uv.y;
    Ray ray = Ray(float3(0., 4., -12), normalize(float3(uv, 1.)));
    float3 col = float3(0.0);
    for (int i=0; i<100; i++) {
        float dist = distToScene(ray);
        if (dist < 0.001) {
            col = float3(1.0);
            break;
        }
        ray.origin += ray.direction * dist;
    }
    float3 n = getNormal(ray);
    output.write(float4(col * n, 1.0), gid);
}
{% endhighlight %}

If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_4.png "4")

Now that we have normals, we can calculate lighting for each pixel in the scene, using the __lighting()__ function. First we need to know the direction to the light (`lightRay`) which we get by normalizing the light position and the current ray. For __diffuse__ lighting we need the angle between the normal and the `lightRay`, that is, the dot product of the two. For __specular__ lighting we need reflections on surfaces, and they depend on the angle we’re looking at. The difference is in this case we first cast a ray into the scene, reflect it from the surface and then we measure the angle between the reflected ray and the `lightRay`. We then take a high power of that value to make it much sharper. Finally we return the combined light:

{% highlight swift %}float lighting(Ray ray, float3 normal, Light light) {
    float3 lightRay = normalize(light.position - ray.origin);
    float diffuse = max(0.0, dot(normal, lightRay));
    float3 reflectedRay = reflect(ray.direction, normal);
    float specular = max(0.0, dot(reflectedRay, lightRay));
    specular = pow(specular, 200.0);
    return diffuse + specular;
}
{% endhighlight %}

Replace the last line in the kernel function with these lines:

{% highlight swift %}Light light = Light(float3(sin(time) * 10.0, 5.0, cos(time) * 10.0));
float l = lighting(ray, n, light);
output.write(float4(col * l, 1.0), gid);
{% endhighlight %}

If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_5.png "5")

Next, shadows! We pretty much use the __shadow()__ function from the first part of this series, with few modifications. We normalize the direction of the light (`lightDir`) and then we just keep updating `distAlongRay` as we march along the ray:

{% highlight swift %}float shadow(Ray ray, Light light) {
    float3 lightDir = light.position - ray.origin;
    float lightDist = length(lightDir);
    lightDir = normalize(lightDir);
    float distAlongRay = 0.01;
    for (int i=0; i<100; i++) {
        Ray lightRay = Ray(ray.origin + lightDir * distAlongRay, lightDir);
        float dist = distToScene(lightRay);
        if (dist < 0.001) {
            return 0.0;
            break;
        }
        distAlongRay += dist;
        if (distAlongRay > lightDist) { break; }
    }
    return 1.0;
}
{% endhighlight %}

Replace the last line in the kernel function with these lines:

{% highlight swift %}float s = shadow(ray, light);
output.write(float4(col * l * s, 1.0), gid);
{% endhighlight %}

If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_6.png "6")

Let's get some `soft shadows` in the scene. In real life, a shadow spreads out the farther it gets from an object. For example, if there is a cube on the floor, at a cube's vertex we get a sharp shadow but farther away from the cube it looks more like a blurred shadow. In other words, we start at some point on the floor, we march towards the light and either hit or miss. Hard shadows are straightforward: we hit something, it's in the shadow. Soft shadows have in-between stages. Update the __shadow()__ function with these lines:

{% highlight swift %}float shadow(Ray ray, float k, Light l) {
    float3 lightDir = l.position - ray.origin;
    float lightDist = length(lightDir);
    lightDir = normalize(lightDir);
    float eps = 0.1;
    float distAlongRay = eps * 2.0;
    float light = 1.0;
    for (int i=0; i<100; i++) {
        Ray lightRay = Ray(ray.origin + lightDir * distAlongRay, lightDir);
        float dist = distToScene(lightRay);
        light = min(light, 1.0 - (eps - dist) / eps);
        distAlongRay += dist * 0.5;
        eps += dist * k;
        if (distAlongRay > lightDist) { break; }
    }
    return max(light, 0.0);
}
{% endhighlight %}

You will notice that we are starting with a white (`1.0`) light this time and we use an attenuator (__k__) to get various (intermediate) values of light. The __eps__ variable tells us how much wider the beam is as we go out into the scene. A thin beam means sharp shadow while a wide beam means soft shadow. We start with a small `distAlongRay` because otherwise the surface at this point would shadow itself. We then travel along the ray as we did for the hard shadows, then we get the distance to the scene, after that we subtract `dist` from `eps` (the beam width) and divide it by `eps`. This gives us the percentage of beam covered. If we invert it (`1 - beam width`) we get the percentage of beam that is in the light. We take the minimum of this new value and `light` to preserve the darkest shadow as we march along the ray. We then again move along the ray and increase the beam width in proportion to the distance traveled and scaled by `k`. If we're past the light, we break out of the loop. Finally, we want to avoid negative values for the light so we return the maximum between __0.0__ and the value of light. Now let's adapt the kernel code to work with the new `shadow()` function:

{% highlight swift %}float3 col = float3(1.0);
bool hit = false;
for (int i=0; i<200; i++) {
    float dist = distToScene(ray);
    if (dist < 0.001) {
        hit = true;
        break;
    }
    ray.origin += ray.direction * dist;
}
if (!hit) {
    col = float3(0.5);
} else {
    float3 n = getNormal(ray);
    Light light = Light(float3(sin(time) * 10.0, 5.0, cos(time) * 10.0));
    float l = lighting(ray, n, light);
    float s = shadow(ray, 0.3, light);
    col = col * l * s;
}
Light light2 = Light(float3(0.0, 5.0, -15.0));
float3 lightRay = normalize(light2.position - ray.origin);
float fl = max(0.0, dot(getNormal(ray), lightRay) / 2.0);
col = col + fl;
output.write(float4(col, 1.0), gid);
{% endhighlight %}

Notice we switched to having a rather white color by default. Then we added a boolean named __hit__ that tells us if we hit the object or not. We determine we have a hit if the distance to scene is within __0.001__. If we didn't hit anything, just color everything in grey, otherwise determine the shadow value. At the end we just add another (fixed) light source in front of the scene so see the shadows in greater detail. If you run the playground now you should see a similar image:

![alt text](https://github.com/MetalKit/images/raw/master/shadows_7.png "7")

To see an animated version of this code, use the `Shadertoy` embedded player below. Just hover over it and click the play button to watch it in action:

<iframe width="740" height="450" frameborder="0" src="https://www.shadertoy.com/embed/XltSWf" allowfullscreen></iframe><br />

The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.

Until next time!
