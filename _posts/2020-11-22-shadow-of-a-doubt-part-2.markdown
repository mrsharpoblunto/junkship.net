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

I've extracted the approximate code I use to do this center correction below. The three main steps are:

1. scaling the new center so its the same distance from the light source as the old center.
2. projecting it into shadow map space to snap it to whole x&y values.
3. reprojecting it back into world space.
4. re-scaling it so its as far away from the light source as the original uncorrected center.

{% highlight c++ %}
constexpr float TexelScale = 2.0f / 1024; // shadow map resolution of 1024px
constexpr float InvTexelScale = 1.0f / TexelScale;

const XMVECTOR newLightDirectionCenter = center - lightPosition;
const XMVECTOR prevLightDirectionCenter =
  previousCenter - lightPosition;

const float newLightDistance =
  XMVectorGetX(XMVector3Length(newLightDirectionCenter));
const float prevLightDistance =
  XMVectorGetX(XMVector3Length(prevLightDirectionCenter));

// move the new center back to the vector normal to the previous position
// (i.e. the plane on which the shadow map is projected so we can see
// where it fits with the previous texel scaling
XMVECTOR scaledNewLightCenter =
  XMVectorScale(center, prevLightDistance / newLightDistance);
scaledNewLightCenter = XMVectorSetW(scaledNewLightCenter, 1.0f);

// project the new center using the previous projection & snap its
// position to a whole texel value, before re-projecting back into world
// space
const XMVECTOR projectedCenter = XMVector4Transform(
  scaledNewLightCenter, cascadeViewProjection);
const float w = XMVectorGetW(projectedCenter);
const float x = floor((XMVectorGetX(projectedCenter) / w) *
  InvTexelScale) * TexelScale;
const float y = floor((XMVectorGetY(projectedCenter) / w) *
  InvTexelScale) * TexelScale;
const float z = XMVectorGetZ(projectedCenter) / w;
XMVECTOR correctedCenter = XMVector4Transform(
  XMVectorSet(x, y, z, 1.0f),
  XMMatrixInverse(nullptr, cascadeViewProjection));
correctedCenter =
  XMVectorScale(correctedCenter, 1.0f / XMVectorGetW(correctedCenter));

// then scale the corrected center out so that its the same distance from
// the light as the original uncorrected center
center =
  lightPosition +
  XMVectorScale(XMVector3Normalize(correctedCenter - lightPosition),
    newLightDistance);
{% endhighlight %}
