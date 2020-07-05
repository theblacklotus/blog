---
layout: post
title:  "Texturedmapped RubberCube"
author: Daniel Collin (emoon/TBL)
author_link: https://twitter.com/daniel_collin
---

This post describes the textured rubbercube effect in Eon.

## Introduction
---

The Texturedmapped RubberCube (TRC from now on) was the first effect I started coding on for Eon. That was long before we had any idea what the demo would actually end up being. Way back when I started working on it I wanted to make some texturemapped effect running at 25 fps on a regular A500. Many of the textured effects I seen in past runs quite a bit slower than that so I would try to push this a bit. I also wanted it to have many colors + a background. What I ended up with is a 16 colors for the cube and 16 colors for the background which was my target. The cube render area I hoped would be around 128x128 and in the end it ended up at 128x105 so pretty close. The reason why I decided to make it a bit smaller will be outlined below.

## The OCS 7 bitplane mode.

One problem when enabling many bit-planes on the Amiga is that the bitplane DMA will steal CPU cycles as more DMA time is required to fetch from memory so the CPU will be stalled more compared to using less bitplanes. There is a bug in the OCS chipset that if you enable 7 bitplanes you get a 5 bitplane mode but no DMA is fetched for the last bitplane. So one may question is how this is actually useful? What we can do is to set the data for the bitplane ourselves at the correct time using the copper and make the colors on at specific time be brought into the 16-31 color range instead of using 0-31. This is how we make the area of the cube use the upper colors instead of using colors 0-16 while paying the DMA cost of having 4 bit-planes active.

