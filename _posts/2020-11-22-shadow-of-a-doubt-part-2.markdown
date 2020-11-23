---
layout: post
title: "Shadow of a doubt - part 2"
date: 2020-11-22 10:54:00
tags:
- graphics
- programming
---

In part 1 I mentioned the problem of shadow stability when using cascading shadow maps, or really any shadow mapping solution where the shadow map moves along with the camera frustum. I didn't really go into detail with the solution though & since its fairly involved I thought I'd follow up with a more detailed explanation.

The instability of shadows happens because the shadow map has limited resolution & when the frustum moves slightly it can result in pixels being quantized into different parts of the shadow map that may have a different visibility state than on the previous frame. You can see this below where I've reduced the resolution of the shadow map and removed all PCF filtering. You can clearly see the edge of the shadow moving as the camera moves. This effect is pretty distracting and ugly - so lets fix it!

###### Unstable shadows that move along with the camera frustum
![Unstable shadows](/assets/images/news/swimming-original.gif)

The way we do this is shown in the diagram below. Essentially we need to correct the position of all cascade shadow map frustums so that the frustum center is always fixed to a texel boundary from the previous frame. We perform this correction on every frame so that the frustum center snaps from boundary to boundary as the camera view vector moves. Doing this ensures that the texel boundaries on the current shadow map line up with the texel boundaries from the previous frames shadow map & eliminates the possibility of pixels for the same world space point being quantized differently by the two shadow maps.

![Texel snapping](/assets/images/news/texel-snapping.png)

You might have noticed that for a perspective shadow map in the diagram above the texel sizes are non-uniform and get larger as you get to the edge of the frustum. This is correct, so its possible for quantization errors to be introduced particularly near the edges of the screen. If you really want to have pixel perfect shadows you'd need to move to a uniform orthographic projection, though in my case orthographic projection doesn't work well for a central point light source like a solar system. As you can see below, once the correction is applied the shadow looks almost completely stable and any minor movement is pretty much unnoticeable (especially so once the shadow map resolution is increased and PCF filtering applied).

###### Stable shadows by snapping frustum movement to shadow map texel positions
![Stable shadows](/assets/images/news/swimming-fixed.gif)

I've extracted the approximate code I use to do this center correction below. The way it works is to project the new center using the previous view-projection matrix, which transforms the x & y coordinates to positions on the shadow map. We then round those x & y coordinates to the nearest texel boundary, then run the projection in reverse (using the inverse of the view-projection matrix) to convert the corrected center back into world space.

{% highlight c++ %}
constexpr float TexelScale = 2.0f / 1024; // shadow map resolution of 1024px
constexpr float InvTexelScale = 1.0f / TexelScale;

// project the new center using the previous projection & snap its
// position to a whole texel value, before re-projecting back into world
// space
const XMVECTOR projectedCenter = XMVector4Transform(
  XMVectorSetW(center, 1.0f), cascadeViewProjection);
const float w = XMVectorGetW(projectedCenter);
const float x = floor((XMVectorGetX(projectedCenter) / w) *
  InvTexelScale) * TexelScale;
const float y = floor((XMVectorGetY(projectedCenter) / w) *
  InvTexelScale) * TexelScale;
const float z = XMVectorGetZ(projectedCenter) / w;
XMVECTOR correctedCenter = XMVector4Transform(
  XMVectorSet(x, y, z, 1.0f),
  XMMatrixInverse(nullptr, cascadeViewProjection));
center =
  XMVectorScale(correctedCenter, 1.0f / XMVectorGetW(correctedCenter));
{% endhighlight %}
