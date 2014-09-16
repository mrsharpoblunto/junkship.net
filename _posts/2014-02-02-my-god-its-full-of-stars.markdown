---
layout: post
title: "My god its full of stars"
date: 2014-02-02 07:02:53
tags:
- graphics
- programming
- directx
- proceduralcontent
---

It has been a long time since I last worked on generating procedural star fields, but I was never particularly happy with the result that I came up with in my [first attempt](http://www.junkship.net/News/2011/12/30/let-men-burn-stars) so when some time presented itself I decided to revisit the problem to see if I could come up with a better solution. It turns out I could, below is an example of what I came up with.

[![image](http://www.junkship.net/Resources/News/rMA18FxTDUCEXHQcErY68Q.png "image")](http://www.junkship.net/Resources/News/385g6y0Bh0qpnJrGXWGuGA.png)

My original method (shown below) was largely based on combining successive layers of Perlin noise, and while this worked ok for the actual stars themselves it usually looked pretty terrible when it came to rendering nebulas and the gas and dust surrounding dense clusters of stars.

[![10-11-2011](http://www.junkship.net/Resources/News/VCODUnjkmkCXS7oVhJM5mw.jpg "10-11-2011")](http://www.junkship.net/Resources/News/x4rC-BorKkyFOEIyEUTvvA.jpg)

The reason for this is that star and galaxy formation is an inherently physical process, shaped by the gravitational interactions of billions of stars and giant clouds of gas and dust. The Perlin noise approach failed to take any of these physical factors into account and as a result it was extremely difficult to tweak it to get the relationship between star and gas density correct, and to get the overall shape and structure of nebulas looking convincing. 

Because of this it seemed obvious that in order to get this right I was going to have to implement some sort of physics based particle system to at least approximate the physics involved. With this in mind I realized that given the number of particles that would have to be simulated (probably millions) that this was going to be impractical to do on the CPU and was going to be a job for the GPU if I wanted to keep the generation time to a minimum.

I ended up writing a Compute shader in which a large number of particles are simultaneously attracted to a handful of attractor particles which move randomly about 3d space. These attractor points are repositioned each frame on the CPU (as there’s only 2 – 3 of them ) and fed into the shader as a buffer. The shader then calculates the acceleration acceleration caused by each particle due the force of each attractor ( proportional to the square of its distance between the attractor and particle ) and updates the particles position. 

The effect of this is a very loose approximation to the force of gravity, but where all the particles can move independently of each other, making it extremely parallelizable. Having multiple attractors is essential in such a simple simulation as having a single attractor results in all particles converging toward a single point, whereas having particles being under the influences of multiple concurrent forces creates interesting and unpredictable patterns. Below is an early rendering showing around 1 million particles being simulated and rendered as point sprites.

[![7-12-2013](http://www.junkship.net/Resources/News/t3uMR6rD50Ws2KoCcQlWvg.jpg "7-12-2013")](http://www.junkship.net/Resources/News/REqWYhEGC0ehB3TBhqqGUw.jpg)

Here's the source code for the compute shader

  <div id="codeSnippetWrapper" style="overflow: auto; cursor: text; font-size: 8pt; border-top: silver 1px solid; height: 400px; font-family: &#39;Courier New&#39;, courier, monospace; border-right: silver 1px solid; width: 97.5%; border-bottom: silver 1px solid; padding-bottom: 4px; direction: ltr; text-align: left; padding-top: 4px; padding-left: 4px; margin: 20px 0px 10px; border-left: silver 1px solid; line-height: 12pt; padding-right: 4px; max-height: 400px; background-color: #f4f4f4">   <div id="codeSnippet" style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">     <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">cbuffer Constants  : <span style="color: #0000ff">register</span>(cb0)</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">{</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    uint GroupDimX;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    uint GroupDimY;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    uint MaxParticles;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">};</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">cbuffer Physics : <span style="color: #0000ff">register</span>(cb1) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">{</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">float</span> FrameTime;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    uint AttractorCount;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">};</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4"><span style="color: #0000ff">struct</span> Particle {</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    float3 CurrentPosition;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    float3 OldPosition;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    float3 Velocity;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">float</span> Scale;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">};</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white"><span style="color: #0000ff">struct</span> Attractor {</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    float3 Position;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    float3 Destination;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">float</span> Velocity;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #0000ff">float</span> Strength;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">float</span> MinAttractorDistance;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">};</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">RWStructuredBuffer&lt;Particle&gt; srcParticleBuffer : <span style="color: #0000ff">register</span>(u0);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">StructuredBuffer&lt;Attractor&gt; attractorsBuffer : <span style="color: #0000ff">register</span>(t0);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">[numthreads(1024, 1, 1)]</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white"><span style="color: #0000ff">void</span> main( uint3 dispatchId : SV_DispatchThreadID )</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">{</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    uint id = dispatchId.x + ( GroupDimX * 1024 * dispatchId.y ) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        + ( GroupDimX * GroupDimY * 1024 * dispatchId.z );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">            </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #008000">//Every thread renders a particle.</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #008000">//If there are more threads than particles then stop here.</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">if</span>(id &lt; MaxParticles){</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        Particle p = srcParticleBuffer[id];</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        float3 a = float3(0.0f,0.0f,0.0f);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        <span style="color: #0000ff">for</span> (uint i=0;i&lt;AttractorCount;++i) {</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">            float3 diff = attractorsBuffer[i].Position - </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                p.CurrentPosition;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">            <span style="color: #0000ff">float</span> distance = length( diff );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">            <span style="color: #0000ff">if</span> ( distance &lt; </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                attractorsBuffer[i].MinAttractorDistance ) {</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                <span style="color: #008000">// make sure particles don't appear inside an </span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                <span style="color: #008000">// attractors min distance. If a particle</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                <span style="color: #008000">// gets inside the min distance, we'll push it </span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                <span style="color: #008000">// to the opposite side of the min sphere</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                <span style="color: #008000">// This reduces large numbers of particles</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                <span style="color: #008000">// converging in a point around an attractor</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                float3 push = diff + </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                    normalize( diff ) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                    * attractorsBuffer[i].MinAttractorDistance;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">                p.OldPosition += push;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                p.CurrentPosition += push;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">            }</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">            a += ( diff * attractorsBuffer[i].Strength ) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">                / (distance * distance);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        }</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        float3 tempPos = 2.0*p.CurrentPosition </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">            - p.OldPosition + a*FrameTime*FrameTime;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        p.OldPosition = p.CurrentPosition;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        p.CurrentPosition = tempPos;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        p.Velocity = p.CurrentPosition - p.OldPosition;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        srcParticleBuffer[id] = p;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    }</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">}</pre>
<!--CRLF--></div>
</div>

The particle simulation when suitably tweaked came up with all sorts of interesting looking nebula type clouds, and worked well for rendering fields of stars. However as you can see in the screenshot above, even with a lot of blur applied to the particles, when rendering a cloud of point sprites they had kind of a lumpy appearance that gave away the fact that the cloud was actually composed of a finite number of particles. I tried all sorts of variations of increased blur, low pass filters, different particle sizes and shapes but the lumpiness was always present to some degree

It took me quite a while to figure out a solution for how to smooth out the particle field and I eventually figured it out after remembering an article I read about the rendering technique used to draw the skyboxes in Homeworld 2 ([http://simonschreibt.de/gat/homeworld-2-backgrounds/](http://simonschreibt.de/gat/homeworld-2-backgrounds/ "http://simonschreibt.de/gat/homeworld-2-backgrounds/")). In that game the backgrounds were all vertex shaded using actual geometry which gave the backgrounds a very smooth almost painted look. 

This got me thinking, could I use vertex shading to try and smooth out the rendered cloud? I started by normalizing all the particle positions so they sat somewhere on a bounding sphere with a radius of 1 unit. Then for each vertex on that sphere I summed up the number of particles within a certain distance and recorded that as the particle density at that vertex. 

![vis](http://www.junkship.net/Resources/News/KVD-ufxZN0C51L1fLoRaVA.png "vis")For example, in this image we can see that the vertex at the center of the red circle has a density of 1 as there is only one particle within the circles radius, whereas the blue vertex has a density of 4 as there are 4 nearby particles

The pixel shader can then render color based on this density, which the graphics card hardware will interpolate for us between the respective vertices for a smooth continuous output. Throwing all these ideas together gave me the following vertex shader (the pixel shader is trivial, it just renders a color whose opacity is based on the vertex.Intensity value and uses the normalized vertex.Velocity vector to lerp between two possible color values )

<div id="codeSnippetWrapper" style="overflow: auto; cursor: text; font-size: 8pt; border-top: silver 1px solid; font-family: &#39;Courier New&#39;, courier, monospace; border-right: silver 1px solid; width: 97.5%; border-bottom: silver 1px solid; padding-bottom: 4px; direction: ltr; text-align: left; padding-top: 4px; padding-left: 4px; margin: 20px 0px 10px; border-left: silver 1px solid; line-height: 12pt; padding-right: 4px; max-height: 400px; background-color: #f4f4f4">
  <div id="codeSnippet" style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">
    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">cbuffer ParamsBuffer: <span style="color: #0000ff">register</span>(cb0)</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">{</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    float4x4 WorldViewProjection;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">float</span> MaxDistance;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #0000ff">int</span> MinThreshold;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">int</span> ParticleCount;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">};</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">StructuredBuffer&lt;Particle&gt; particleBuffer : <span style="color: #0000ff">register</span>(u0);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">Nebula_VSOutput main(VSInput input)</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">{</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    Nebula_VSOutput output;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    float4 worldPosition = mul(float4(input.Position,1.0f), </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        WorldViewProjection);</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    output.LocalPosition = input.Position;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    output.Position = worldPosition.xyww;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    output.Velocity = float3( 0.0f, 0.0f, 0.0f );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #0000ff">int</span> intensity = 0;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #008000">// find how many particles are within a threshold distance</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #008000">// of this vertex</span></pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    <span style="color: #0000ff">for</span> (<span style="color: #0000ff">int</span> i = 0; i &lt; ParticleCount; i++)</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    {</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        Particle p = particleBuffer[i];</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        <span style="color: #0000ff">float</span> distance = length( normalize( p.CurrentPosition ) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">            - input.Position );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        intensity += distance &lt;= MaxDistance ? 1 : 0;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">        output.Velocity += p.Velocity;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    }</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    output.Velocity = normalize( output.Velocity );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">    output.Intensity = saturate( ( intensity - MinThreshold ) </pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">        / (<span style="color: #0000ff">float</span>)ParticleCount );</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">&#160;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: #f4f4f4">    <span style="color: #0000ff">return</span> output;</pre>
<!--CRLF-->

    <pre style="border-top-style: none; overflow: visible; font-size: 8pt; font-family: &#39;Courier New&#39;, courier, monospace; width: 100%; border-bottom-style: none; color: black; padding-bottom: 0px; direction: ltr; text-align: left; padding-top: 0px; border-right-style: none; padding-left: 0px; margin: 0em; border-left-style: none; line-height: 12pt; padding-right: 0px; background-color: white">}</pre>
<!--CRLF--></div>
</div>

Turns out it worked exactly as I had hoped! the rendered clouds were now perfectly smooth provided that the sphere used to render the skybox had a sufficient polygon count. It also had the ancillary benefit of requiring far less particles for an equivalently detailed cloud which helped reduce the simulation time. Below are a couple of different procedural star field renderings, each field is rendered after around 60 physics iterations and contains around 2 million particles for the point sprite stars, and around fifty thousand particles for the nebula clouds. A full skybox is rendered in 300 layers with one layer per frame ( so as to not adversely affect the frame rate of other rendering ) which ends up taking around 5 seconds.

[![image](http://www.junkship.net/Resources/News/RLLR_soBfEWUe7XkkUTIWQ.png "image")](http://www.junkship.net/Resources/News/zSZbPkYKOku3SPS8d5dYRw.png)

[![image](http://www.junkship.net/Resources/News/leER1UvM20aph6JUbLxwEw.png "image")](http://www.junkship.net/Resources/News/XVlc-gbtBkuZUOhXsIfKEQ.png)