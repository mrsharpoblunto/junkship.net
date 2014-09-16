---
layout: post
title: "Porting to Directx 11 - A Code Odyssey"
date: 2012-11-04 06:11:49
---

Well its finally done. Junkship and the [MGDF framework](https://github.com/mrsharpoblunto/MGDF) has been fully ported from DirectX 9 to DirectX 11, and while it wasn’t something that I can say I particularly enjoyed at times – I can see in hindsight that the pain endured was worthwhile. Along the way I learnt a whole lot and recorded a number of the issues I encountered, which I will outline here for the benefit of any other wary travellers who are considering going down the same path.

First things first, DirectX 11 is VERY different to DirectX 9. I skipped the whole DirectX 10 thing, so I can’t say for certain but it looks that for the most part, porting from DX10 to DX11 is relatively trivial as most of the API’s are the same (just with all the 10’s changed to 11’s). Unfortunately this is not the case with DX9 to DX11 (and so too with DX9 to DX10) – Almost every API is different, and there isn’t always a clear upgrade path for certain DX9 features. In addition, DX11 is almost universally more verbose than DX9, but in exchange you get much more flexibility. DX9 is very restrictive in certain areas, mostly due to its support for the old fixed function pipeline, and in DX11 these restrictions are gone in favor of having to write more boilerplate.

&#160;

### Domain shaders and Tessellation are awesome

Ok, technically this has nothing to do with porting, but its cool so I’m going to include it. I implemented displacement mapping (bump mapping is just faking it… displacement mapping is the real deal) for the asteroid rendering in Junkship using the new tessellation features in DX11. Hover over the screenshot to see the difference displacement mapping makes to the asteroid geometry (though it does bring less capable GPU’s to their knees)
  <div style="width: 464px; height: 349px" id="hover">![image](http://www.junkship.net/Resources/News/zYgifXKvukWwpVF7NCObag.png "image")![image](http://www.junkship.net/Resources/News/DnoeXkPfVEKrLjcwAEr1fw.png "image")</div> <script type="text/javascript">
document.getElementById("hover").onmouseover = function() {
	document.getElementById("img1").style.display = 'inline';
	document.getElementById("img2").style.display = 'none';
}
document.getElementById("hover").onmouseout = function() {
	document.getElementById("img1").style.display = 'none';
	document.getElementById("img2").style.display = 'inline';
}
</script>  

&#160;

### Sane initialization and feature support

One of the main problems with DX9 is that there was no clear minimum feature set required by hardware, this meant you had to laboriously check for support for pretty much every specific feature of the API you wanted to use. With DX10, this is no longer an issue as there is a clearly defined minimum feature set. In DX11 the ability to target a number of different feature levels has been added, so If you have a DX9 game and you are worried about using DX11 because you will leave users with old machines out in the cold – Don’t worry, you can use the DX11 API but specify that you want to support DX9 hardware (The table of feature levels and the minimum requirements are listed [here](http://msdn.microsoft.com/en-us/library/windows/desktop/ff476876%28v=vs.85%29.aspx))

In addition, the code for initializing DirectX and managing transitions from windowed to fullscreen mode has been improved. You no longer have to worry about resizing the window rectangle explicitly (Though you will still have to deal with resizing the back buffer)

&#160;

### No more lost devices (Great for GPU generated procedural resources)

In DX11 you no longer have to worry about the device getting lost. This means that you no longer have to regenerate resources from the&#160; D3DPOOL_DEFAULT (In fact the notion of putting resources in a DEFAULT or MANAGED pool no longer exists) every time the window is resized, or the user presses alt-tab. This is particularly important for Junkship as I generate large amounts of textures procedurally on the GPU, so in DX9 these resources had to be D3DPOOL_DEFAULT which meant that to prevent having to regenerate the resources every time the device became lost I had to copy each generated resource into a D3DPOOL_MANAGED resource. The problem is is that copying back to the managed pool in DX9 is very slow at the best of times, and ridiculously slow at the worst as GPU’s usually batch instructions and run independently of whatever the CPU is up to, so trying to copy a resource from the GPU before it has finished generating causes a pipeline stall which kills performance. The tricky thing is that you can’t be sure on the CPU side when the GPU has finished, so you have to make a best guess and wait and hope that you don’t stall the pipeline when you try to copy. Not having to do this pointless copying DOUBLED the speed of Junkship’s procedural texture generation.

&#160;

### Multithreading friendly

DX9 by default isn’t thread safe, and enabling multithreading support wasn’t recommended as the performance suffered considerably, luckily DX11 is considerably more multithreading friendly. Instead of executing instructions using the d3d device like in DX9, in DX11 you can construct any number of device contexts which can batch up commands on different threads and submit them to the GPU separately (the commands are still serialized on the GPU side, but this is largely transparent to the API user). Another nice feature is that you can run D3D on a thread other than the main window thread (nice if you want to have a separate render thread from the input poll loop or simulation thread) which wasn’t possible in DX9.

&#160;

### D3D Debug mode is useful and not a global setting

In DX9, there's a switch in the DirectX control panel to enable D3D debug mode, which unfortunately applies to every D3D application on your system. Unfortunately I would regularly forget to switch it back to retail mode and would wonder why games I was playing would perform poorly or have strange graphical issues. In fact during one period in which I was addicted to [Galactic civilizations 2](http://www.galciv2.com/index.aspx) the in game minimap started appearing blank. Amazingly I put up with this for months until one day I noticed that D3D was running in debug mode. Returning it to retail mode fixed the issue &lt;/facepalm&gt;. DX11 has a much more sane approach of letting you define whether debug mode is enabled for specific applications. Also from my experience the debug messages in DX11 seem to be much easier to follow than their more cryptic DX9 counterparts.

&#160;

### D3DX is dead, so is the D3DX effects framework – Don’t use them

The venerable D3DX library has been officially deprecated in Windows 8, and so while they are still available for use with DX11, I wouldn’t recommend it. There are decent alternatives for most of the functionality provided in D3DX in the [DirectXTex](http://directxtex.codeplex.com/) and [DirectXTK](http://directxtk.codeplex.com/) libraries, and all the math related functionality in D3DX now exist in the built in xnamath (if you’re using the DirectX SDK) or directX math (Which is just a renamed version of xnamath if you’re using the Win8 SDK) libraries.&#160; In addition the effects framework is no longer a core part of D3DX and is instead supplied as a separate source code library, however once again I wouldn’t recommend using it. There are three main reasons, firstly is that D3DX style .fx files are now deprecated by the HLSL compiler, secondly the D3DX effects framework makes use of shader reflection and run time compilation which is verboten if you want to write a metro style game, and thirdly – its just not that good. With a little work you should be able to write an effects system that is more tailored to your games needs and is also more flexible. I wrote my own effects management system and now I get all sorts of nice benefits like altering shaders while the program is running and seeing the rendering change accordingly – I might write some more about this in future if I get time.

&#160;

### Compile your shaders offline

If you weren’t already, compile your shaders offline using the FXC compiler supplied in the SDK (the flags etc. for making it work can be found [here](http://msdn.microsoft.com/en-us/library/windows/desktop/bb509709%28v=vs.85%29.aspx)). Going forward runtime shader compilation will not be supported for Windows 8 Metro applications and should only ever be done for debug builds on your dev machine. I prefer to precompile my shaders even when developing as it lets me know instantly if my shaders are valid without having to boot up the game. To do this I wrote a small program that runs as a post build step which resolves all shader last modified times + the last modified times of any #imported shader fragments invokes FXC on them (I found this easier than configuring custom build tasks for the hlsl files in visual studio as it provides a centralized place to setup what flags etc. I want to compile with and to apply any custom processing required as a part of my effects framework). This is something that I might post up to GitHub in the future once I tidy it up.

&#160;

### No point sprites – Use instancing or geometry shaders 

In Junkship I used the DX9 point sprites feature to render the background starfield, in DX11 the point sprites feature has been removed so to get the equivalent functionality you have two choices – Object instancing or geometry shaders. To get a point sprite effect using object instancing involves creating two vertex buffers – the first being a quad, the other being a matrix transform defining how to orient that quad for every sprite you wish to display. To render the sprites, you would then use a pixel shader which draws a texture over the quad, and using the [ID3D11DeviceContext::DrawIndexedInstanced](http://msdn.microsoft.com/en-us/library/windows/desktop/ff476410%28v=vs.85%29.aspx) method will ensure that the sprite will be drawn for every orientation specified in the second vertex buffer. The other option (the one I chose) was to use a Geometry shader. Geometry shaders are a relatively new type of shader (introduced in DX10) which allow the GPU to generate new vertices in addition to those supplied via the usual vertex buffers. To render billboard style point sprites (sprites that always face the camera) you will need to supply all the positions of the point sprites in a vertex buffer, then in a geometry shader create 4 vertices about this position that exist on a plane that the position-&gt;camera vector is normal to (the HLSL code for this is shown below) Once this is done you should be able to use the same pixel shader as used by the instanced object technique.
  <div style="border-bottom: silver 1px solid; text-align: left; border-left: silver 1px solid; padding-bottom: 4px; line-height: 12pt; background-color: #f4f4f4; margin: 20px 0px 10px; padding-left: 4px; width: 97.5%; padding-right: 4px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; height: 261px; max-height: 200px; font-size: 8pt; overflow: auto; border-top: silver 1px solid; cursor: text; border-right: silver 1px solid; padding-top: 4px" id="codeSnippetWrapper">   <div style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px" id="codeSnippet">     <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum1">   1:</span> [maxvertexcount(4)] <span style="color: #008000">// point sprite quad, so 4 vertices</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum2">   2:</span> <span style="color: #0000ff">void</span> main(point Starfield_VSOutput input[1],inout TriangleStream&lt;Starfield_GSOutput&gt; stream)</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum3">   3:</span> {</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum4">   4:</span>     </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum5">   5:</span>     float3 right;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum6">   6:</span>     float3 up;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum7">   7:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum8">   8:</span>     <span style="color: #0000ff">if</span> (abs(input[0].LocalPosition.y) &lt; 0.5)</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum9">   9:</span>     {</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum10">  10:</span>         right = cross(UP,input[0].LocalPosition);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum11">  11:</span>         up = cross(input[0].LocalPosition,right);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum12">  12:</span>     }</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum13">  13:</span>     <span style="color: #0000ff">else</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum14">  14:</span>     {</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum15">  15:</span>         <span style="color: #008000">//calculate the positions using absolute right instead</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum16">  16:</span>         <span style="color: #008000">//of absolute UP when the position is near the north/south</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum17">  17:</span>         <span style="color: #008000">//poles to avoid errors in the cross product with near</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum18">  18:</span>         <span style="color: #008000">//parallel vectors</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum19">  19:</span>         up = cross(input[0].LocalPosition,RIGHT);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum20">  20:</span>         right = cross(input[0].LocalPosition,up * -1);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum21">  21:</span>     }</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum22">  22:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum23">  23:</span>     up = normalize(up);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum24">  24:</span>     right = normalize(right);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum25">  25:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum26">  26:</span>     <span style="color: #0000ff">float</span> intensity =  saturate(1.0 - length(input[0].LocalPosition));</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum27">  27:</span>     <span style="color: #0000ff">float</span> scale =  MinScale + (intensity * (MaxScale - MinScale));</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum28">  28:</span>     float3 v1 = input[0].LocalPosition    + (right * scale) - (up * scale);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum29">  29:</span>     float3 v2 = input[0].LocalPosition  + (right * scale) + (up * scale);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum30">  30:</span>     float3 v3 = input[0].LocalPosition    - (right * scale) - (up * scale);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum31">  31:</span>     float3 v4 = input[0].LocalPosition    - (right * scale) + (up * scale);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum32">  32:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum33">  33:</span>     Starfield_GSOutput output;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum34">  34:</span>     output.UV = float2(1.0,1.0);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum35">  35:</span>     output.Intensity = intensity;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum36">  36:</span>     output.LocalPosition = input[0].LocalPosition;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum37">  37:</span>     output.Position = mul(float4(v1,1.0),WorldViewProjection);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum38">  38:</span>     stream.Append(output);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum39">  39:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum40">  40:</span>     output.UV = float2(1.0,0.0);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum41">  41:</span>     output.Intensity = intensity;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum42">  42:</span>     output.LocalPosition = input[0].LocalPosition;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum43">  43:</span>     output.Position = mul(float4(v2,1.0),WorldViewProjection);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum44">  44:</span>     stream.Append(output);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum45">  45:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum46">  46:</span>     output.UV = float2(0.0,1.0);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum47">  47:</span>     output.Intensity = intensity;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum48">  48:</span>     output.LocalPosition = input[0].LocalPosition;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum49">  49:</span>     output.Position = mul(float4(v3,1.0),WorldViewProjection);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum50">  50:</span>     stream.Append(output);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum51">  51:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum52">  52:</span>     output.UV = float2(0.0,0.0);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum53">  53:</span>     output.Intensity = intensity;</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum54">  54:</span>     output.LocalPosition = input[0].LocalPosition;    </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum55">  55:</span>     output.Position = mul(float4(v4,1.0),WorldViewProjection);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum56">  56:</span>     stream.Append(output);</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum57">  57:</span> }</pre>
<!--CRLF--></div>
</div>

### Direct2D/DirectWrite doesn’t work with DX11

some reason Microsoft decided not to add support in Windows 7 for&#160; Directwrite/Direct2d access to DX11 surfaces (apparently this is remedied in Windows 8), which means that to render text one either has to rely on sprite fonts or (warning WTF approaching) create a DX10.1 device that uses DirectWrite + Direct2D to render text to a shared backbuffer with DX11. Instead of dealing with this mess I used the [FW1FontWrapper](http://fw1.codeplex.com/) library, which has met my needs thus far.

&#160;

### Ditch DirectInput (If you haven’t already)

I have been dragging my heels on this for a number of years as I know the use of DirectInput is deprecated and not recommended, so I finally bit the bullet and switched to using RawInput. Despite the (horrendously bad) documentation on how to use it, it actually proved to be considerably simpler than the DirectInput code that it replaced. One thing to note with RawInput is that if you want responsive input you have to ensure that your main window thread processes messages quickly and doesn’t experience any delays. For this reason I moved all rendering into a separate thread (the game simulation was already) as I didn’t want the input latency to be tied to the rendering frame rate. So now the window thread is now only handling input and windows API messages, and rendering and game simulation are run on two separate threads. (see [here](https://github.com/mrsharpoblunto/MGDF/blob/master/Matchstick/src/core/MGDFD3DAppFramework.cpp) for how I set up the sim/rendering/input threads in the MGDF framework that Junkship runs on)

&#160;

### Visual studio 2012 Graphical debugging is great

I also upgraded to Visual Studio 2012, which has a great new feature for debugging DirectX applications. Its pretty much the old PIX tool that we all know and love that has been given a serious facelift (PIX was great functionally, but the UI was truly horrible) and integrated into Visual Studio. For me this is a killer feature over VS2010 (Once you get over the ALL CAPS MENUS which everyone seems to be enraged about – Its kind of weird but doesn’t really annoy me to be honest)

[![image](http://www.junkship.net/Resources/News/vLZQJTqjsUOjjM2Z0nT-aw.png "image")](http://www.junkship.net/Resources/News/J9F9a0Yd0E2li-XyjE0-Lw.png)

So now Junkship is fully ported (and leaner and faster than ever) I can get back to doing the fun stuff again ![Smile](http://www.junkship.net/Resources/News/K_7n0OI_cEq4Mpud1dNAOg.png)