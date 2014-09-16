---
layout: post
title: "Beachballs in space"
date: 2012-06-25 01:06:07
tags:
- graphics
- proceduralcontent
---

Heres a quick post on how I elaborated on a technique I first saw used by [Paul Whigham](http://johnwhigham.blogspot.com/2011/11/gas-giants.html) to simulate whirling vortices on gas planets. I wont go into huge depth explaining the technique as he has already done a great job of doing so, but essentially the technique involves creating a number of cones which protrude from the center of a sphere outward. Each of these cones represents a conical surface detail. One then renders a [voronoi](http://en.wikipedia.org/wiki/Voronoi_diagram) diagram on the surface of the sphere mapping out for each pixel which cone is closest and encoding this into the textures RGBA data.

&#160;

[![image](http://www.junkship.net/Resources/News/l4PN-OqvukOCc20mDKJ0Dw.png "image")](http://www.junkship.net/Resources/News/2y3UmdweF0y0akK1yMXJ2Q.png)

<font size="2">Beachballs in space - An example of a cube mapped voronoi diagram</font>

&#160;

<font size="3">Once this map is rendered, when rendering the object in the pixel shader one can then determine the strength of the nearest cone by sampling the voronoi map texture, and use this to offset the texture lookup to produce a whirl effect.</font>

&#160;

[![image](http://www.junkship.net/Resources/News/8mpJocuSU0enI2nEgIqj8Q.png "image")](http://www.junkship.net/Resources/News/gAp_qj_M40iNI1l6ebuPZw.png)

<font size="2">A gas planet without diffuse lighting to better illustrate the whirl surface effect</font>

&#160;

Whigham doesn’t go into detail as to how to calculate this offset, so in the interest of helping others out, here’s the relevant pieces of HLSL that I came up with.

&#160;
  <pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//rotates a vector about an arbitrary axis</span>
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span>3 RotateAboutAxis(in <span style="color: #0000ff">float</span>3 v,in <span style="color: #0000ff">float</span> angle,in <span style="color: #0000ff">float</span>3 axis)
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">{
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">	<span style="color: #008000">//http://en.wikipedia.org/wiki/Rodrigues%27_rotation_formula</span>
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">	<span style="color: #0000ff">return</span> (v * <span style="color: #0000ff">cos</span>(angle)) + (cross(axis,v)*<span style="color: #0000ff">sin</span>(angle)) + (axis * dot(axis,v)*(1-<span style="color: #0000ff">cos</span>(angle)));
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">}
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"></pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">...
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"></pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//sample the voronoi map in object space</span>
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span>4 vornoiSample = texCUBE(vornoiSampler,input.LocalPosition);
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"></pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//decode the properties from the voronoi map</span>
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span> radius = vornoiSample.a;
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span>3 whirlPosition = vornoiSample * 2 - 1.0;
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span> strength = <span style="color: #0000ff">abs</span>(whirlPosition.z);
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//recreate the z value for the cone axis</span>
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">whirlPosition.z = <span style="color: #0000ff">sqrt</span>(1.0f - whirlPosition.y*whirlPosition.y - whirlPosition.x*whirlPosition.x) * sign(whirlPosition.z);
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"></pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//find the distance between the current pixel and the intersection point of the cone axis and sphere	</span>
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span>3 difference = whirlPosition-input.LocalPosition;
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span> distance = length(difference);
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">	
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//calculate the strength of the rotation at the given distance from the cone axis.</span>
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//the strength diminishes by distance squared from the axis outward</span>
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span> attenuation = saturate(1.0 - (distance/radius));
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span> theta = (strength * MAX_ROTATION) * (attenuation * attenuation);
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">	
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px">input.Normal = normalize(input.Normal);
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #008000">//adjust the final cubemap texture lookup by rotating the lookup by theta about the cones axis.</span>
</pre><pre style="background-color: #f4f4f4; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"><span style="color: #0000ff">float</span>3 adjustedPosition = whirlPosition - (theta&gt;0 ? RotateAboutAxis(difference,theta,whirlPosition) : difference);
</pre><pre style="background-color: #ffffff; margin: 0em; width: 100%; font-family: consolas,&#39;Courier New&#39;,courier,monospace; font-size: 12px"></pre></pre>

&#160;

I also found that this technique can be used for any roughly circular surface details. One such is case is generating impact craters on the surface of rocky planets. The general process is the same, except that instead of rotating texture lookups, one uses the voronoi map to adjust the heightmap data to produce circular surface indentations.

&#160;

[![image](http://www.junkship.net/Resources/News/Os6oTMav4Ea7j5MRxgQGlw.png "image")](http://www.junkship.net/Resources/News/WY0-d8lDykqqOpiNIKQG2A.png)

<font size="2">The craters in this image are produced by using a voronoi map to adjust the heightmap.</font>

&#160;

There are a bunch of other uses I can see this technique being useful for such as adjusting the strength of night time lightmaps on planetary surfaces to create cities, or varying diffuse colors on procedural planetary textures to create ‘biome’ type effects.