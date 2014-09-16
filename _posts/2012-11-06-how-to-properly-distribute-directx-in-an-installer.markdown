---
layout: post
title: "How to properly distribute DirectX in an installer"
date: 2012-11-06 06:11:12
tags:
- programming
- directx
---

I see threads all the time where people complain about game installers that have DirectX install wizards that pop up mid install, or about the size of the end user redistributable bloating their game installer. I understand why this is a problem, there’s lots of advice and documentation telling you what NOT to do when distributing DirectX (Don’t just include the dlls in your install folder!), but not much in the way of what you SHOULD do. So after seeing another iteration of this discussion on this recent [Reddit thread](http://www.reddit.com/r/gaming/comments/12o9fd/bundling_directx_9_with_your_game_the_right_way/), I decided I’d write this post so that those who are unsure of how to redistribute DirectX in their own games installer will be able to get it right (BTW - the installer in the linked thread is doing it wrong despite the thread title).

### &#160;

### Step 1 – Create a minimal installation package

Most Windows installs already have DirectX installed by default, Windows XP SP2+ includes DirectX 9.0c, Windows Vista includes DirectX 9&amp;10 (and SP2 includes DirectX 11), and Windows 7 includes 9,10&amp;11 right out of the box. So really all you have to do to ensure that all prerequisites are installed correctly is to ensure that any dll’s you use that are part of the DirectX SDK and NOT part of the core DirectX install are installed.

The first step in doing this is to go to the DirectX SDK install directory (usually something like “C:\Program Files (x86)\Microsoft DirectX SDK (June 2010)”) and go into the Redist folder. Create a new folder somewhere else on disk and copy the following files from the Redist folder into it

*   DXSetup.exe*   DSETUP.dll*   dsetup32.dll*   dxupdate.cab  

You should now have a folder that looks something like the following

[![image](http://www.junkship.net/Resources/News/l72Guhea20KlBCsHAT4LKA.png "image")](http://www.junkship.net/Resources/News/kIdg3s0_aUyXzMLwjoBD1Q.png)   

### Step 2 – Determine what D3D libraries you are using

Now you need to find what D3D libraries your solution is actually using. If you’re not sure (and your using c++) In Visual Studio, go to project properties and you probably have these listed under the ‘Additional Dependencies’ field in the Linker-&gt;Input window. Take note of any .lib files starting with d or x (e.g. xinput.lib or d3dx9.lib). 

Now go back to the DirectX SDK Redist folder and for each lib you identified, grab the most recent copy of the corresponding .cab file. For example, if you are using d3dx9.lib, then (assuming you are using the June 2010 SDK) you would grab Jun2010_d3dx9_43_x86.cab (also assuming your game is compiled as a 32 bit application – if not, go with the x64 versions of the same .cab). Similarly if you were using xinput.lib, you would grab APR2007_xinput_x86.cab. Copy all the selected .cab files across to your install package folder we created in the previous step.

You should now have a folder that looks something like the following

[![image](http://www.junkship.net/Resources/News/7PwNowwC00SeBvCZvqKysg.png "image")](http://www.junkship.net/Resources/News/8BJo8Jz19EmEbQrGRMDUhA.png)   

### Step 3 – Bundle these files in your installer and execute DXSETUP *silently*

The files you identified above are all that are required to install the DirectX prerequisites on any users machine. Note how its only a fraction of the size of the full end user redistributable. Now make sure to include all those files in your installer, then at install time, extract them to a temp folder and run DXSETUP.exe with the /silent argument like so

dxsetup.exe /silent

It is critical that you use the /silent argument as this prevents the DirectX installer UI popping up during your own install. Don’t worry if the user already has the prerequisites, the dxsetup program is smart enough to not install anything that is already installed.

Here’s an example from Junkship’s [nsis](http://nsis.sourceforge.net/Main_Page) installer script which does the DirectX install
  <div style="border-bottom: silver 1px solid; text-align: left; border-left: silver 1px solid; padding-bottom: 4px; line-height: 12pt; background-color: #f4f4f4; margin: 20px 0px 10px; padding-left: 4px; width: 97.5%; padding-right: 4px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; max-height: 200px; font-size: 8pt; overflow: auto; border-top: silver 1px solid; cursor: text; border-right: silver 1px solid; padding-top: 4px" id="codeSnippetWrapper">   <div style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px" id="codeSnippet">     <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum1">   1:</span> ;--------------------------------</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum2">   2:</span> ; install directx11 update</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum3">   3:</span> Section <span style="color: #006080">&quot;Install Directx 11 June 2010 update&quot;</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum4">   4:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum5">   5:</span>     SetOutPath $TEMP\Junkship</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum6">   6:</span>     File dxredist\*.cab</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum7">   7:</span>     File dxredist\*.dll</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum8">   8:</span>     File dxredist\DXSETUP.exe</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum9">   9:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum10">  10:</span>     ExecWait <span style="color: #006080">'&quot;$TEMP\Junkship\dxsetup.exe&quot; /silent'</span></pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum11">  11:</span>     SetOutPath $TEMP</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum12">  12:</span>     RMDIR /r $TEMP\Junkship\dxredist</pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: white; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum13">  13:</span>  </pre>
<!--CRLF-->

    <pre style="border-bottom-style: none; text-align: left; padding-bottom: 0px; line-height: 12pt; background-color: #f4f4f4; margin: 0em; border-left-style: none; padding-left: 0px; width: 100%; padding-right: 0px; font-family: &#39;Courier New&#39;, courier, monospace; direction: ltr; border-top-style: none; color: black; border-right-style: none; font-size: 8pt; overflow: visible; padding-top: 0px"><span style="color: #606060" id="lnum14">  14:</span> SectionEnd</pre>
<!--CRLF--></div>
</div>

### Step 4 – Profit!

That’s it! (though if you are using DirectX 11 and want to support windows Vista users, you may need to do some additional prerequisite checking as detailed in [this article](http://msdn.microsoft.com/en-US/library/ee416644.aspx))

**NOTE:** Most of the information included from this post was sourced from [this MSDN article](http://msdn.microsoft.com/en-US/library/ee416644.aspx) if you want to know more