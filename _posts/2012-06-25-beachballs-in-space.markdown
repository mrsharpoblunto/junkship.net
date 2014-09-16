---
layout: post
title: "Beachballs in space"
date: 2012-06-25 01:06:07
tags:
- graphics
- proceduralcontent
---

Heres a quick post on how I elaborated on a technique I first saw used by [Paul Whigham](http://johnwhigham.blogspot.com/2011/11/gas-giants.html) to simulate whirling vortices on gas planets. I wont go into huge depth explaining the technique as he has already done a great job of doing so, but essentially the technique involves creating a number of cones which protrude from the center of a sphere outward. Each of these cones represents a conical surface detail. One then renders a [voronoi](http://en.wikipedia.org/wiki/Voronoi_diagram) diagram on the surface of the sphere mapping out for each pixel which cone is closest and encoding this into the textures RGBA data.

![image](/assets/images/news/2y3UmdweF0y0akK1yMXJ2Q.jpg)

<font size="2">Beachballs in space - An example of a cube mapped voronoi diagram</font>

Once this map is rendered, when rendering the object in the pixel shader one can then determine the strength of the nearest cone by sampling the voronoi map texture, and use this to offset the texture lookup to produce a whirl effect.

![image](/assets/images/news/gAp_qj_M40iNI1l6ebuPZw.jpg)

<font size="2">A gas planet without diffuse lighting to better illustrate the whirl surface effect</font>

Whigham doesn’t go into detail as to how to calculate this offset, so in the interest of helping others out, here’s the relevant pieces of HLSL that I came up with.

{% higlight c %}
//rotates a vector about an arbitrary axis

float3 RotateAboutAxis(in float3 v,in float angle,in float3 axis)
{
    //http://en.wikipedia.org/wiki/Rodrigues%27_rotation_formula
    return (v * cos(angle)) + (cross(axis,v)*sin(angle)) + (axis * dot(axis,v)*(1-cos(angle)));
}

...

//sample the voronoi map in object space
float4 vornoiSample = texCUBE(vornoiSampler,input.LocalPosition);

//decode the properties from the voronoi map
float radius = vornoiSample.a;
float3 whirlPosition = vornoiSample * 2 - 1.0;
float strength = abs(whirlPosition.z);

//recreate the z value for the cone axis
whirlPosition.z = sqrt(1.0f - whirlPosition.y*whirlPosition.y - whirlPosition.x*whirlPosition.x) * sign(whirlPosition.z);

//find the distance between the current pixel and the intersection point of the cone axis and sphere    
float3 difference = whirlPosition-input.LocalPosition;
float distance = length(difference);

//calculate the strength of the rotation at the given distance from the cone axis.
//the strength diminishes by distance squared from the axis outward
float attenuation = saturate(1.0 - (distance/radius));
float theta = (strength * MAX_ROTATION) * (attenuation * attenuation);

input.Normal = normalize(input.Normal);
//adjust the final cubemap texture lookup by rotating the lookup by theta about the cones axis.

float3 adjustedPosition = whirlPosition - (theta>0 ? RotateAboutAxis(difference,theta,whirlPosition) : difference);
{% endhighlight %}

I also found that this technique can be used for any roughly circular surface details. One such is case is generating impact craters on the surface of rocky planets. The general process is the same, except that instead of rotating texture lookups, one uses the voronoi map to adjust the heightmap data to produce circular surface indentations.

![image](/assets/images/news/WY0-d8lDykqqOpiNIKQG2A.jpg)

<font size="2">The craters in this image are produced by using a voronoi map to adjust the heightmap.</font>

There are a bunch of other uses I can see this technique being useful for such as adjusting the strength of night time lightmaps on planetary surfaces to create cities, or varying diffuse colors on procedural planetary textures to create ‘biome’ type effects.
