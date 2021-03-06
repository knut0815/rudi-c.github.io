---
title: Consistent distance fields for ray marching
date: 2017-06-21
disqus: y
---

$$
\newcommand{\norm}[1]{\left\lVert #1 \right\rVert}
$$

Introduction
------------

The classical way of writing a ray tracer represents objects in your scene with a triangle mesh. You then shoot some light rays out of your camera and do some algebra to see if it intersects with any triangles.

In recent years, an alternative to triangle-based ray tracing has emerged. While slightly less general, it's fast enough to ray-trace scenes in real-time. It's been popular in the hobbyist community for being able to produce cool results with few lines of code. The key is to use something called a ["distance field"](//www.iquilezles.org/www/material/nvscene2008/rwwtt.pdf).

A distance function \\( D(x,y,z) \\) tells you, for any point \\( (x,y,z) \\) in space, how far the *closest* object is. It doesn't tell you what direction, just the distance to the closest object. Using such a function, however, makes ray marching really easy. You start the ray at the position of the camera. You calculate \\( d = D(x,y,z) \\). This tells you that you can move the ray forward by distance \\( d \\) without hitting any object. Repeat this process until you get within some epsilon distance of the object.

<center><img src="/images/2017/raymarching.svg" width="400"/></center>

The code for this process is equally simple:

```glsl
float castRay(vec3 pos, vec3 rayDir) {
    float dist = 0.0;

    for (int i = 0; i < ITERATIONS; i++)
    {
        float res = distanceFn(pos + rayDir * dist);
        if (res < EPSILON || dist > MAX_DIST) break;
        dist += res;
    }

    return dist;
}
```

How do you find these distance functions? Constructing them for some primitives is very simple. Here is a sphere and the distance function of a sphere:

<center><img src="/images/2017/sphere.png" width="300"/></center>

```glsl
float distanceSphere(vec3 pos, float radius)
{
    return length(pos) - radius;
}
```

You can find many more examples of primitive shapes, as well as how to combine them and modify them, on [Inigo Quilez's website](//www.iquilezles.org/www/articles/distfunctions/distfunctions.htm).

It can be difficult, however, to create distance functions for complex shapes using only primitive operations. What people will often do in practice is to create functions that give you an estimate of the distance to the closest object. But regardless of the nature of the estimate, points where the distance function \\( d(x, y, z) = 0 \\) represent the surface of the object (think for a second why this is true).

For example, I've obtained the funky-looking shape on the left by using the distance function the right.

<center><img src="/images/2017/funkysphere.png" width="300"/></center>

```glsl
float distanceFunkySphere(vec3 pos, float radius, float frequency)
{
    float noise = sin(pos.x*frequency)*sin(pos.y*frequency)*sin(pos.z*frequency);
    return length(pos) - radius + noise;
}
```

Where the original distanceSphere function gave me the exact distance to the closest object, the new `distanceFunkySphere` function doesn't do that. It gives me an estimate to the distance of *some surface* (that's no longer just a sphere). But my ray marching code never needed the exact distance to begin with, since it gets closer to the surface step by step.

Finally, you can render this super fast because each of these ray marching procedures on distance functions can be evaluated independently for each pixel on the screen (you just change the ray direction). This is the kind of work that the GPU is best at!

If this is new to you so far and you find this interesting, I recommend taking a look at cool demos on [Shadertoy](//www.shadertoy.com) and exploring [Inigo Quilez's website](//www.iquilezles.org/index.html) who gets a lot of credit for popularizing this technique. If you're already familiar with distance field ray marching, read on, as I'm going to get into some technical details that are important for producing good distance functions.

Overstepping
------------

In the example above, I multiplied some sin functions and added it to a distance function of a sphere to get a sphere with bubbles. If you're acquainted with the hobbyist community that uses distance functions, something about it should feel familiar. You do some random math operations, get something physically incorrect, stick in some arbitrary constants for adjustment, wave your hands and say "hey, the result looks nice!".

A friend of mine, [Veit Heller](//veitheller.de/) coined a good term for this: **voodoo math**.

Getting a physically incorrect result isn't always a problem. You could, for example, divide a distance function by 2. It'll never answer the question "how far is the closest point?" correctly, but at worst it'll take twice as many steps for ray marching to hit the surface of the object.

Problems occur when your distance function gives you a result that's too large. Then you might take a ray marching step past the surface of your object.

<center><img src="/images/2017/raymarching_overstep.svg" width="400"/></center>

This kind of overstepping will either fail to render thin objects (jump past it) or render the inside of an object, leading to weird results.

To see how this is important, consider the animation below. On the left side, I'm ray marching 25% faster than the safe "speed limit" (more on that in a second). On the right side, I have the reference, correct image. If you squint your eyes and look carefully at the surface as it's moving, you should see some "ripples" that appear at random times. That's caused by overstepping.

<center><iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/ldlfRj?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe></center>

<!--
#define MAX_DIST 4.
#define EPSILON 0.0005
#define ITERATIONS 256

// Uncomment to plot the distance to the object
// instead of using regular light-based shading
// #define RENDER_DISTANCE

// Uncomment to artificially magnify differences
// between the left side with artefacts and the
// reference right side by five times
// #define MAKE_OBVIOUS

#define NOISE_PROPORTION 0.75
#define NOISE_FREQUENCY 50.
//#define NOISE_GRADIENT_MAGNITUDE (1. * NOISE_FREQUENCY) // max
#define NOISE_GRADIENT_MAGNITUDE (0.88 * NOISE_FREQUENCY) // 95th percentile
#define SPHERE_GRADIENT_MAGNITUDE 1.
#define OVERSTEP_FACTOR 1.25

// Distance to a sphere centered at the origin
float sdSphere(vec3 pos, float radius) {
    return length(pos) - radius;
}

float noise(vec3 pos) {
    return sin(pos.x)*sin(pos.y)*sin(pos.z);
}

// Distance to a sphere with sinusoidal perturbations
// centered at the origin.
// Copied over from https://www.shadertoy.com/view/Xds3zN
float sdWeirdSphere(vec3 pos) {
    return mix(sdSphere(pos, 0.2) / SPHERE_GRADIENT_MAGNITUDE,
               noise(pos * NOISE_FREQUENCY) / NOISE_GRADIENT_MAGNITUDE,
               NOISE_PROPORTION);
}

float distanceFn(vec3 pos, bool artefacts) {
    if (artefacts) {
        return OVERSTEP_FACTOR * sdWeirdSphere(pos);
    } else {
        return sdWeirdSphere(pos);
    }
}

// Copied over from https://www.shadertoy.com/view/Xds3zN
vec3 calcNormal(vec3 pos, bool artefacts) {
    vec2 e = vec2(1.0,-1.0)*0.5773*0.0005;
    return normalize( e.xyy*distanceFn( pos + e.xyy, artefacts ) +
                      e.yyx*distanceFn( pos + e.yyx, artefacts ) +
                      e.yxy*distanceFn( pos + e.yxy, artefacts ) +
                      e.xxx*distanceFn( pos + e.xxx, artefacts ) );
}

// Ray marching using distance-estimation
// Adapted from https://www.shadertoy.com/view/Xds3zN
float castRay(vec3 pos, vec3 rayDir, bool artefacts) {
    float dist = 0.;

    for (int i = 0; i < ITERATIONS; i++)
    {
        // More precision when the object is closer
        float precis = EPSILON * dist;
        float res = distanceFn(pos + rayDir * dist, artefacts);
        if (res < precis || dist > MAX_DIST) break;
        dist += res;
    }

    return dist;
}

// Copied over from https://www.shadertoy.com/view/Xds3zN
// setCamera(origin, target, rotation)
mat3 setCamera(vec3 ro, vec3 ta, float cr)
{
    vec3 cw = normalize(ta-ro);
    vec3 cp = vec3(sin(cr), cos(cr),0.0);
    vec3 cu = normalize( cross(cw,cp) );
    vec3 cv = normalize( cross(cu,cw) );
    return mat3( cu, cv, cw );
}

// Copied over from https://www.shadertoy.com/view/Xds3zN
vec3 shade(vec3 camera, vec3 rayDir, float dist, bool artefacts) {
    vec3 rayPos = camera + rayDir * dist;
    vec3 normal = calcNormal(rayPos, artefacts);
    vec3 ref = reflect(rayDir, normal);

    float m = 65.;
    vec3 col = 0.45 + 0.35*sin( vec3(0.05,0.08,0.10)*(m-1.0) );

    // Lighting
    vec3  lig = normalize( vec3(-0.4, 0.7, -0.6) );
    float amb = clamp( 0.5+0.5*normal.y, 0.0, 1.0 );
    float dif = clamp( dot( normal, lig ), 0.0, 1.0 );
    float bac = clamp( dot( normal, normalize(vec3(-lig.x,0.0,-lig.z))), 0.0, 1.0 )*clamp( 1.0-rayPos.y,0.0,1.0);
    float dom = smoothstep( -0.1, 0.1, ref.y );
    float fre = pow( clamp(1.0+dot(normal,rayDir),0.0,1.0), 2.0 );
    float spe = pow(clamp( dot( ref, lig ), 0.0, 1.0 ),16.0);

    vec3 lin = vec3(0.3);
    lin += 1.30*dif*vec3(1.00,0.80,0.55);
    lin += 2.00*spe*vec3(1.00,0.90,0.70);
    lin += 0.40*amb*vec3(0.40,0.60,1.00);
    lin += 0.50*dom*vec3(0.40,0.60,1.00);
    lin += 0.50*bac*vec3(0.25,0.25,0.25);
    lin += 0.25*fre*vec3(1.00,1.00,1.00);
    col = col*lin;

    col = mix( col, vec3(0.8,0.9,1.0), 1.0-exp( -0.0002*dist*dist*dist ) );

    return vec3( clamp(col,0.0,1.0) );
}

vec3 distanceShade(float dist, float minDist, float maxDist) {
    float normalized = (dist - minDist) / (maxDist - minDist);
    if (normalized < 1. / 4.) {
        float inRange = normalized * 4.;
        return vec3(inRange, 0., 0.);
    } else if (normalized < 2. / 4.) {
        float inRange = (normalized - 1. / 4.) * 4.;
        return vec3(1. - inRange, inRange, 0.);
    } else if (normalized < 3. / 4.) {
        float inRange = (normalized - 2. / 4.) * 4.;
        return vec3(0., 1. - inRange, inRange);
    } else {
        float inRange = (normalized - 3. / 4.) * 4.;
        return vec3(0., 0., 1. - inRange);
    }
}

vec4 render(vec2 uv, bool artefacts) {
    vec3 camera = vec3(cos(iTime / 4.), 0.0, sin(iTime / 4.)) * 0.75;
    vec3 target = vec3(0.0);

    // Camera-to-world transformation
    mat3 transform = setCamera(camera, target, 0.0);

    // Origin is now at the center of the screen (given camera position);
    vec3 rayDirection = transform * normalize(vec3(uv, 2.0));

    float dist = castRay(camera, rayDirection, artefacts);

    if (dist > MAX_DIST) {
        return vec4(uv, 0., 1.0);
    } else {
        #ifdef RENDER_DISTANCE
            return vec4(distanceShade(dist, 0.475, 0.8), 1.0);
        #else
            return vec4(shade(camera, rayDirection, dist, artefacts), 1.0);
        #endif
    }
}

// fragColor: 0 to 1, starting at bottom-left corner
void mainImage(out vec4 fragColor, in vec2 fragCoord) {
    // Divide by two since we split the screen
    vec2 splitResolution = vec2(iResolution.x / 2., iResolution.y);
    float minDimension = min(splitResolution.x, splitResolution.y);
    vec2 uv = (fragCoord - splitResolution / 2.) / minDimension * 2.;
    if (fragCoord.x < iResolution.x / 2.) {
        #if defined(MAKE_OBVIOUS) && !defined(RENDER_DISTANCE)
            vec4 wrong = render(uv, true);
            vec4 correct = render(uv, false);
            fragColor = wrong + (wrong - correct) * 5.;
        #else
            fragColor = render(uv, true);
        #endif
    } else {
        fragColor = render(uv - vec2(iResolution.x / minDimension, 0.0), false);
    }
}
-->

I used this particular shape as an example since it's used in the [primitives demo](//www.shadertoy.com/view/Xds3zN). I noticed that the rippling happens as the object gets close to the camera.

Preventing overstepping
-----------------------

How do you guarantee that overstepping won't happen? A useful heuristic is to look at how fast the function changes. If the *magnitude of the gradient of the function is less or equal to one everywhere that it's defined*, then you will never overstep.

To get an intuition of how this is true, consider the one-dimensional case, where \\( D(x) \\) is the distance to a point at \\( x = 0 \\). Then the line \\( D(x) = x \\) is the maximum allowed value of \\( D(x) \\) and has gradient (derivative) equal to 1 everywhere except at \\( x = 0 \\) where it's not defined. If the gradient can only be smaller, \\( D(x) \\) can also only be smaller.

<center><img src="/images/2017/1d_distance_ok.svg" width="250"/></center>

This is a rather strong condition. You could have functions that are consistent distance fields (that do not overstep) and that have a large gradient in some small area that doesn't matter, like the one below.

<center><img src="/images/2017/1d_distance_large.svg" width="250"/></center>

In that case, dividing by the gradient magnitude would be unnecessarily conservative. However, if we start adding it to other distance functions, than that small area with large gradient suddenly starts to matter.

<center><img src="/images/2017/1d_distance_problem.svg" width="600"/></center>

In the case of our sin noise function, that is what we do.

Anyway, what is the gradient of our noise function? Recall that it's defined as:

$$
n(x, y, z) = (\sin{wx})(\sin{wy})(\sin{wz})
$$

Then \\( \nabla n = (w\cos{wx}\sin{wy}\sin{wz}, w\sin{wx}\cos{wy}\sin{wz}, w\sin{wx}\sin{wy}\cos{wz}) \\). Next we find an upper bound for the gradient magnitude.

$$
\begin{align*}
\norm{\nabla n} = & \sqrt{ w^2\cos^2{wx}\sin^2{wy}\sin^2{wz} + w^2\sin^2{wx}\cos^2{wy}\sin^2{wz} + w^2\sin^2{wx}\sin^2{wy}\cos^2{wz} } \\
                = & w \sqrt{ \cos^2{wx}\sin^2{wy}\sin^2{wz} + \sin^2{wx}\cos^2{wy}\sin^2{wz} + \sin^2{wx}\sin^2{wy}\cos^2{wz} } \\
                  & \text{use trig identity:} \cos^2 = 1 - \sin^2 \\
                = & w \sqrt{ (1 - \sin^2{wx})\sin^2{wy}\sin^2{wz} + \sin^2{wx}\cos^2{wy}\sin^2{wz} + \sin^2{wx}\sin^2{wy}\cos^2{wz} } \\
                = & w \sqrt{ \sin^2{wy}\sin^2{wz} + \sin^2{wx}(\cos^2{wy}\sin^2{wz} + \sin^2{wy}\cos^2{wz} - \sin^2{wy}\sin^2{wz}) } \\
              \le & w \sqrt{ \sin^2{wy}\sin^2{wz} + (\cos^2{wy}\sin^2{wz} + \sin^2{wy}\cos^2{wz} - \sin^2{wy}\sin^2{wz}) } \\
                = & w \sqrt{ \cos^2{wy}\sin^2{wz} + \sin^2{wy}\cos^2{wz} } \\
                = & w \sqrt{ (1 - \sin^2{wy})\sin^2{wz} + \sin^2{wy}\cos^2{wz} } \\
                = & w \sqrt{ \sin^2{wz} + \sin^2{wy}(\cos^2{wz} - \sin^2{wz}) } \\
              \le & w \sqrt{ \sin^2{wz} + (\cos^2{wz} - \sin^2{wz}) } \\
                = & w \sqrt{ \cos^2{wz} } \\
              \le & w \sqrt{ 1 } \\
                = & w \\
\end{align*}
$$

So the gradient of \\( n(x, y, z) \\) is at most \\( w \\), the noise frequency. That means that \\( \frac{n(x,y,z)}{w} \\) is a distance function guaranteed not to overstep.

But wait, the distance function that we used was \\( D(x,y,z) = D_{sphere}(x,y,z) + D_{noise}(x,y,z) \\)!

The gradient of \\( D_{sphere} \\) is exactly 1 everywhere (exercise for the reader - this is also true for all exact distance functions). We can use the triangle inequality to get that \\( \norm{\nabla D(x,y,z)} <= \norm{\nabla D_{sphere}(x,y,z)} + \norm{\nabla D_{noise}(x,y,z)} = 1 + w \\). So it suffices to divide \\( D(x,y,z) \\) by \\( (1 + w) \\). Now, using that distance function, we are guaranteed not to overstep.

In practice, we adjust the strength of the noise component by using linear interpolation \\( D(x,y,z) = (1 - ratio) * D_{sphere}(x,y,z) + ratio * D_{noise}(x,y,z) / w \\). In GLSL, this can be done with the `mix` function.

Useful properties of gradients and gradient maximums
----------------------------------------------------

Linearity: \\( \nabla(\alpha F(\vec{v}) + \beta G(\vec{v})) = \alpha \nabla F(\vec{v}) + \beta \nabla G(\vec{v}) \\)

Product rule: \\( \nabla(F(\vec{v}) G(\vec{v})) = F(\vec{v}) \nabla G(\vec{v}) +  G(\vec{v}) \nabla F(\vec{v}) \\)

Chain rule: \\( \nabla(g(F(\vec{v}))) = g'(F(\vec{v})) \nabla F(\vec{v}) \\)

Minimum: \\( \nabla (\min(F(\vec{v}), G(\vec{v}))) \le \max(\nabla F(\vec{v}), \nabla G(\vec{v})) \\)

Maximum: \\( \nabla (\max(F(\vec{v}), G(\vec{v}))) \le \max(\nabla F(\vec{v}), \nabla G(\vec{v})) \\)

Absolute value: \\( \nabla (abs(F(\vec{v}))) = \nabla F(\vec{v}) \\).

Triangle inequality: If \\( \vec{a} = \vec{b} + \vec{c} \\), then \\( \norm{\vec{a}} \le \norm{\vec{b}} + \norm{\vec{c}} \\).

Fast ray marching
-----------------

The method we used to find a scaling factor for our distance function uses the maximum of the magnitude of the gradient as well as the triangle inequality. It gives us a conservative estimate, based on the potential worst cases. This means that we are probably overestimating the scaling factor and marching slower than we need to. Can we do better? See my next blog post.
