---
layout: post
title: "Shadow of a doubt"
date: 2020-2-15 10:54:00
tags:
- graphics
- programming
---

You might think that realtime shadows are a largely solved problem. Most games have them, and for the most part they look great - but this hides the fact that aside from some recent advances in realtime ray-tracing (RTX etc.) the state of the art for realtime shadown rendering is an ingenious, but in practical terms terribly flawed technique known as the shadow map. I knew eventually I'd have to build in shadowing support into my engine, but what I didn't realize was that shadow mapping techniques are really optimized for usecases where your viewpoint is close to a solid ground plain with a directional lightsource projecting down from above. Space represents a really challenging environment as you don't have any ground plain, you have a point light source (a star) - not a directional one, and there are potentially huge distances between shadow casters and shadow receivers.
TODO explain shadow mapping basics... quick diagram


Soft shadows
============
The problem with classic shadow maps is that they cast hard edged shadows, so over time a number of techniques have been developed to try and achieve more natural looking shadows

VSM - Variance shadow maps. These produce really bad 'haloing' artifacts when you have overlapping shadow casters. In a crowded asteroid field, you frequently have more than one asteroid blocking a light source, so this is a non-starter

ESM - Exponential shadow maps. These have a nasty edge case that as long as you have a ground plain and don't have too many overlapping shadow casters you probably wouldn't notice. In a space asteroid field, these assumptions are false much more often leading to frequent unsightly artifacts (TODO see image)

PCF - Percentage closer filtering - A brute force technique to just take multiple samples around a pixel and average the results to smooth out the edge of the shadows. While its not very elegant, its simple and gets the job done so thats what I ended up using

Shadow map resolution
=====================
The next problem you run into is that since shadow mapping is a screen space technique, the shadow map has a finite resolution, so you end up having to balance between the resolution of shadows & the draw distance of those shadows as the map has a limited amount of world space that it can cover and for a larger worldspace target, each pixel in the shadow map needs to cover a larger area leading to pixellated shadows. This is where most of the tweaking and hacks around effective use of shadow maps comes from - optimizing the space in the shadow map so that you get the best resolution for shadows close to the camera combined with an acceptable shadow draw distance.

Directional vs point lights
===========================
As I mentioned - most uses of shadow maps involve directional lights. The reason for this is that you can use an orthographic projection to render the shadow depth map, which results in a nice uniform resolution of the shadow depth map. Using a point light involves having to do a perspective projection of the depth map that results in higher resolution of shadow samples toward the center of the shadow map. Unfortunately in the kind of space environment I want to render we have a point light i.e. a star in the center of the solar system, so we have to use a perspective projection. Now to be totally accurate you would need to do a cubemap projection and get a depthmap from the stars position covering all possible viewer positions, but this is impractical as the shadow resolution would end up being far to low & you'd be rendering depth maps for objects that are so far from the viewer as to be non-visible anyway. So in order to increase the shadow map resolution to practical levels we cheat by only recording the depth map that lies within the visible area of the screen - known as the camera frustrum.

TODO frustrum/depth map sketch

This gives us the best use of our available shadow map space as we're only rendering depth values for the area that covers the visible screen - however, even in this case the limited resolution of the shadow map means that up-close shadows still look very blocky. To fix this we can rely on another brute-force technique - known as cascading shadow maps (CSM). Just render a bunch of depth maps starting from a small one close to the camera and expanding in size along the view vector, then when rendering the shadows, pick the shadow depth samples from the highest available depth map at that point in the view space. Each cascade overlaps the others, so we can get a bit of extra resolution by not partitioning strictly by the distance along the view vector, but by checking if the world space position hits a position that the depth map covered - that way we can aggressively prioritize using the closer & therefore more detailed shadow maps. Theres also some art to picking how to split the cascades & how many to do - I use a logarithmic distribution, favoring more maps closer to the camera for better up close resolution at the expense of less detail further away & found 6 cascades to be a good balance between quality and performance

TODO show comparison of naive vs optimized picking.

TODO CSM  picking colors.

Theres also another problem that this introduces, since the depth map is constantly moving along with the camera view, the shadows can appear to move or 'swim' when you move the camera. For this you have to go to some considerable lengths to 'snap' movement in the depth map vectors to whole texel multiples (i.e. you need to calculate the world space size of a texel at the point of the view vector and only allow the depth map source to move in increments of that - this doesn't completely remove shadow movement (as perspective depth maps have non-linear texel sizes, but it keeps things largely stable in the center of the viewport).

Distance between casters and recievers
======================================
All the hacks above get us to a point where we get pretty nice looking dynamic shadows for objects up close, but as things get further away the lower resolution of the shadow map makes them less and less effective. This is a problem in space scenes where there is typically very large distances involved and we don't really want shadows from small objects like asteroids projecting gigantic shadows onto planets half way across the solar system. To solve this I decided to treat very large and very small objects as separate as far as the lighting and shadowing was concerned. And because the large objects have very predictable geometry I could come up with some very geometry specific shadowing techniques that allow for much better results than a generic technique like shadow mapping.

All planets are spheres & its mathematically relatively simple to do a sphere vector intersection test so you can analytically determine if any planet is in the shadow of any other planet. You can also determine the distance of the intersection from the center of the shadow caster sphere to the edge & can therefore trivially do soft shadows from any given planet to another. 

TODO sketch of planet geometric shadow

Similarly the shadows on a planetary ring system can be determined with some basic geometry. In that case you want to see if the vector at any point in the ring plane passes through the planet sphere or not. Finally you can also cast shadows from the ring plane down onto the planet itself by doing a plane intersection test from a point on the planet sphere through to the lightsource, determine the radius of the intersection from the origin & then lookup the ring texture to see the alpha value of the ring at that point.

