---
layout: post
title: "Let men burn stars"
date: 2011-12-30 01:12:33
---

As promised last post, here’s a brief dev diary entry on creating random starfields and nebulas. After generating a passable looking planet the next step is to generate some cool looking&#160; space for the planet to exist inside, and for that you need a couple of things. 

&#160;

Firstly you need a starfield. for this I created a spherical point cloud of star sprites centred about the camera. Getting a good even distribution of stars was a little trickier than I first thought it would be but I found a good algorithm for evenly distributing points about a sphere as shown below

&#160;
  <div class="csharpcode">   <pre class="alt">StarParticle* p = 0;</pre>

  <pre>HR(field-&gt;VertexBuffer-&gt;Lock(0, 0, (<span class="kwrd">void</span>**)&amp;p, D3DLOCK_DISCARD));</pre>

  <pre class="alt"><span class="kwrd">for</span> (<span class="kwrd">int</span> i=0;i&lt;starCount;++i)</pre>

  <pre>{</pre>

  <pre class="alt">    StarParticle sp;</pre>

  <pre>    <span class="kwrd">float</span> u = (((rand() % 10000)/(<span class="kwrd">float</span>)10000) - 0.5) * 2.0;</pre>

  <pre class="alt">    <span class="kwrd">float</span> theta = ((rand() % 10000)/(<span class="kwrd">float</span>)10000) * 2 * D3DX_PI;</pre>

  <pre>    </pre>

  <pre class="alt">    <span class="kwrd">float</span> distance = (rand() % (<span class="kwrd">int</span>)(maxDistance-minDistance))+minDistance;</pre>

  <pre>    <span class="rem">//uniform distribution of points about a sphere http://mathworld.wolfram.com/SpherePointPicking.html</span></pre>

  <pre class="alt">    sp.Position.x = (minDistance+distance) * sqrt(1-pow(u,2)) * cos(theta);</pre>

  <pre>    sp.Position.y = (minDistance+distance) * sqrt(1-pow(u,2)) * sin(theta);</pre>

  <pre class="alt">    sp.Position.z = (minDistance+distance) * u;</pre>

  <pre>    sp.Size = (<span class="kwrd">float</span>)starDesc.Width * (1-(distance/maxDistance));</pre>

  <pre class="alt">    p[i] = sp;</pre>

  <pre>}</pre>

  <pre class="alt">HR(field-&gt;VertexBuffer-&gt;Unlock());</pre>
</div>
<style type="text/css">

.csharpcode, .csharpcode pre
{
	font-size: small;
	color: black;
	font-family: consolas, "Courier New", courier, monospace;
	background-color: #ffffff;
	/*white-space: pre;*/
}
.csharpcode pre { margin: 0em; }
.csharpcode .rem { color: #008000; }
.csharpcode .kwrd { color: #0000ff; }
.csharpcode .str { color: #006080; }
.csharpcode .op { color: #0000c0; }
.csharpcode .preproc { color: #cc6633; }
.csharpcode .asp { background-color: #ffff00; }
.csharpcode .html { color: #800000; }
.csharpcode .attr { color: #ff0000; }
.csharpcode .alt 
{
	background-color: #f4f4f4;
	width: 100%;
	margin: 0em;
}
.csharpcode .lnum { color: #606060; }</style>

&#160;

I also generate the star sprites procedurally by setting a size and start color in the center of a square, then as distance from the center increases, exponentially decaying the opacity. This produces sprites that look like this

&#160;

[![image](http://www.junkship.net/Resources/News/mu4JPOwpBUy1YBNERygVlQ.png "image")](http://www.junkship.net/Resources/News/LiiJIkJ6X0OUbsnMNhttsA.png) 

&#160;

When put together you get something like this (Note: I render the starfield’s into a cubemap skybox so that there’s no need to regenerate the skybox every single frame)

&#160;

[![image](http://www.junkship.net/Resources/News/g5E5QNtzJkK-GxYuqWkDFw.png "image")](http://www.junkship.net/Resources/News/XOxnCfQH0Ea1V-f0gaaMGQ.png) 

&#160;

Now this is all well and good, but it does look very dull, and its also very hard to orient yourself as there are no recognizable landmarks. To fix this I added a number of extra detail layers. The first of these was two smooth background noise layers to add some lightness and variety to the background. 

&#160;

[![image](http://www.junkship.net/Resources/News/Mf3dHFOOJ0au8zopyMtirQ.png "image")](http://www.junkship.net/Resources/News/QW80ZUcxvUiM_B3bL9sTBQ.png) 

&#160;

While this looks a little better its still very dull so I added a few layers of ridged Perlin noise to simulate gas nebula clouds (you can create ridges in noise by picking an offset and subtracting the raw noise value from that offset, bigger or smaller offsets create different ridging patterns). I also added some post-process bloom to give the sky a little more glow.

&#160;

[![image](http://www.junkship.net/Resources/News/kND0tnZcVkyC3Er6uxv7cA.png "image")](http://www.junkship.net/Resources/News/KXwtueAYKE-ArbcmRGrXfA.png) 

&#160;

Finally I added a few layers of brighter stars of different colors and extra background star layers that were masked such that they were most visible around the nebula clouds to give the appearance of greater star density in the nebulas.

&#160;

[![image](http://www.junkship.net/Resources/News/Nm0pyV_4IECev7SRGKZ2ag.png "image")](http://www.junkship.net/Resources/News/siP9uOTvOkSx7TKxIzhyZA.png) 

&#160;

The results aren’t going to fool anyone into thinking its a Hubble photo, but I think they do look quite good. One thing that I haven’t yet implemented is a random palette picker to pick varied, yet complimentary colors for the nebula clouds to further increase variety. The palette picker can wait however as the next entry will be on the subject of generating random asteroid fields :)