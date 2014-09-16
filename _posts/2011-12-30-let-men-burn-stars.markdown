---
layout: post
title: "Let men burn stars"
date: 2011-12-30 01:12:33
---

As promised last post, here’s a brief dev diary entry on creating random starfields and nebulas. After generating a passable looking planet the next step is to generate some cool looking&#160; space for the planet to exist inside, and for that you need a couple of things. 

&#160;

Firstly you need a starfield. for this I created a spherical point cloud of star sprites centred about the camera. Getting a good even distribution of stars was a little trickier than I first thought it would be but I found a good algorithm for evenly distributing points about a sphere as shown below
 
{% highlight c++ %}

StarParticle* p = 0;
HR(field->VertexBuffer->Lock(0, 0, (void**)&p, D3DLOCK_DISCARD));
for (int i=0;i<starCount;++i)
{
    StarParticle sp;
    float u = (((rand() % 10000)/(float)10000) - 0.5) * 2.0;
    float theta = ((rand() % 10000)/(float)10000) * 2 * D3DX_PI;

    float distance = (rand() % (int)(maxDistance-minDistance))+minDistance;
    //uniform distribution of points about a sphere http://mathworld.wolfram.com/SpherePointPicking.html
    sp.Position.x = (minDistance+distance) * sqrt(1-pow(u,2)) * cos(theta);
    sp.Position.y = (minDistance+distance) * sqrt(1-pow(u,2)) * sin(theta);
    sp.Position.z = (minDistance+distance) * u;
    sp.Size = (float)starDesc.Width * (1-(distance/maxDistance));
    p[i] = sp;
}
HR(field->VertexBuffer->Unlock());
{% endhighlight %}

I also generate the star sprites procedurally by setting a size and start color in the center of a square, then as distance from the center increases, exponentially decaying the opacity. This produces sprites that look like this

![image](/assets/images/news/LiiJIkJ6X0OUbsnMNhttsA.jpg) 

When put together you get something like this (Note: I render the starfield’s into a cubemap skybox so that there’s no need to regenerate the skybox every single frame)

![image](/assets/images/news/XOxnCfQH0Ea1V-f0gaaMGQ.jpg) 

Now this is all well and good, but it does look very dull, and its also very hard to orient yourself as there are no recognizable landmarks. To fix this I added a number of extra detail layers. The first of these was two smooth background noise layers to add some lightness and variety to the background. 

![image](/assets/images/news/QW80ZUcxvUiM_B3bL9sTBQ.jpg) 

![image](/assets/images/news/KXwtueAYKE-ArbcmRGrXfA.jpg) 

Finally I added a few layers of brighter stars of different colors and extra background star layers that were masked such that they were most visible around the nebula clouds to give the appearance of greater star density in the nebulas.

![image](/assets/images/news/siP9uOTvOkSx7TKxIzhyZA.jpg) 

The results aren’t going to fool anyone into thinking its a Hubble photo, but I think they do look quite good. One thing that I haven’t yet implemented is a random palette picker to pick varied, yet complimentary colors for the nebula clouds to further increase variety. The palette picker can wait however as the next entry will be on the subject of generating random asteroid fields :)
