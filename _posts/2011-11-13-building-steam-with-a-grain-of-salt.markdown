---
layout: post
title: "Building steam with a grain of salt"
date: 2011-11-13 01:11:27
tags:
- graphics
- design
- proceduralcontent
---

Its been an insanely long time since this website has had an update, but I guess its better late than never. Anyways, about 3 months ago Junkship underwent a significant design refresh that was prompted by a few key factors. The first of these was the realization that as a small indie team, the scope and in particular the content creation required for the original Junkship concept was not really feasible. The second factor was a general disillusionment with hand crafted narratives and stories in games, and the desire to try a novel approach to narrative, and the third and final factor was an awesome game concept demo by Luke Smith that signalled a new and interesting direction that we could take Junkship in.

While I’m not going to go into any detail as to the specifics of the new gameplay (much of it is in flux anyway), one new aspect is that the game world will now be much more of a sandbox with an emphasis on procedurally generated content. The advantage of this are twofold, a sandbox arrangement allows the player the freedom to create their own narratives and stories, and procedural content allows the experiences of the player somewhat unique as well as reducing the content creation workload for us :)

So in order to facilitate this vision I’ve started off down the path of generating procedural space environments, and in fact the purpose of this post is to show the progress I’ve made thus far. The end goal is to procedurally generate solar systems with tens of planets, hundreds of moons and many thousands of asteroids, each of which have settlements and trade routes etc between them. While this is a fair way off, this end goal is simply the aggregation of a number of base content generators (a starfield generator, an earthlike planet generator, a rocky planet generator, an asteroid field generator, etc.)

I decided to start with the task of writing a generator for procedural earthlike planets. The basic idea was to use improved Perlin noise to generate a heightmap, which combined with a color gradient and a known sealevel I could produce a decent approximation of a planet with water oceans and continents. Below is the first iteration of this idea. To produce the terrain I used two layers of blended noise, the first was for the base continent terrain, and the second was a layer of ridged noise to produce mountain range like features. I also settled on using HLSL shaders on the GPU to do the heavy lifting as far as generating the terrain as the CPU proved too slow (10 seconds + per planet on the CPU compared with milliseconds on the GPU).

![09-09-2011](/assets/images/news/Rj51m4d_Y0mokbpCn_DrsQ.jpg) 

While the above screenshot is all well and good its got a long way before it looks properly earthlike. The next step was to calculate terrain normals from the generated heightmap to allow bumpmapping, as well as calculating specular terms for the water and terrain. Generating the normal map was relatively simple given that I already had all the height data and just had to record the height differences between adjacent texels to get what I wanted. The hardest part getting the bump mapping to actually work was due to me foolishly recording the bumpmap vectors in world space rather than the usual tangent space.

![19-09-2011](/assets/images/news/o9KjlmettkCi-bCjl_Kulw.jpg) 
   After producing a bumpmap like the one above (I also included a specular map in the alpha channel of the bumpmap, which you can’t see in the above image) I could now produce planets that looked like the one below.     

![20-09-2011](/assets/images/news/6jm0qQjrAki2vyg9GDwzvA.jpg) 

The main problem I now had to overcome was to make sure the terrain had realistic variations in color. After staring at pictures of earth for hours I identified a few main techniques I could use in order to improve the terrain. The most obvious addition I could make was to ensure that the poles and areas with high altitude were snowy (and have a high specular value for more shininess). A more subtle variation on this is that forests and jungles are more common at lower altitudes and closer to the equator, with deserts being more common at higher altitudes and more extreme latitudes. Finally I could also vary terrain color based on the steepness of the terrain itself. By using two square color gradient textures and sampling/blending between them using the heightmap/normal map/texel latitude I was able to produce much better looking terrain.
   ![22-09-2011]/assets/images/news/5piRjUYQn0qGt2E18NF21Q.jpg)       

The main feature I was now missing was clouds. I implemented these using two noise fields which I would additively blend together using a varying offset to simulate changing clouds over time. Getting the noise to look right was relatively easy, however I found the clouds really didn’t look right until they cast shadows on the terrain below them, once I did this the results were really good (especially when you see them changing with time).
   ![03-10-2011](/assets/images/news/HxdbxEgQ8kiHpSp32PRsKQ.jpg)     

One last thing I wanted to add was city lights on the night side of the planet (assuming of course that the planet is inhabited) to do this I used another noise map similar to the cloud layer and prioritized the intensity to areas more typically inhabited (i.e. near the coast and in the more comfortable latitudes).
   ![04-10-2011](/assets/images/news/KloZ6-JfWEGg4C3LlGSn2g.jpg)     

After putting it all together…
   ![1-11-2011](/assets/images/news/ssOIe3LsC06UOdTcrzeJ3Q.jpg)    Satisfied with the results so far, my next challenge was to generate random starfields and nebulas (of which the previous screenshot shows an early version), but that will have to wait for another post.   
