---
layout: post
title:  "Making Eon: Introduction"
author: Daniel Collin (emoon/TBL)
author_link: https://twitter.com/daniel_collin
---

About 5 years ago during the Revision Amiga Demo competition I made a comment about the OCS entries were of quite low quality and that we should do something about it (now days it has improved a lot!). This was really the start of Eon and the first commit to the Github repository was made 2015-04-06 just to get the basic structure into place. Early on I set up some directions I wanted to achieve for the demo:

* Target: A500 512K+512K
* No greetings, no standard text scrollers, no "classic" effects such as glenz vectors.
* Coherent design with a red thread in the whole demo. No "random" static images that doesn't fit the design.
* Two disks. Switch to next floppy should be seamless (no "insert disk 2 screen")
* None or minimal amount of "loading" parts.
* Demo should start as soon as possible (no loading demo screen)
* Music shouldn't be a classic A500 demo track.
* Allow music to vary more thanks to sample streaming.
* Reserve 200K of chipmem for sample buffer.
* Vocal samples in the music.
* 3D scenes that were more detailed than done before on the platform at good frame-rate (25 fps)
* Demo parts should run in 50 or 25 fps (never lower) on target hardware.
* Make a good looking demo, not a good looking demo for being A500.
* Willing to trade disk space for better performance.

Looking back at this list we achieved all of it except the vocals part for the music. We had them in there for a long time but in the end we had to cut them because of space and timing issues. The only good place to have them would be in the transition to disk 2 and it ended up being quite messy to get that to work in a good way so in the end we decided it wasn't worth the effort.
I knew early on that some people wouldn't like this demo but to be honest I really didn't care and still don't. I wanted to make something a bit different to the norm and I think we made it there.

## Vision for the demo

There will be a separate post on how the art direction and such for the demo was done but I will touch on it here a bit as well. Calladin made the whole direction and almost all the graphics for the demo and a really big part of how the demo looks is a result of his work.

Here is a collection of screenshots from some of the rough sketches to just get a sense of the demo. Something we also did that was very helpful during production was to have a previz of the demo. A previz is a video where we had a version of the music + visuals. In the beginning it was pretty much just still images but we kept updating this with mock-up effects and latest version of the music. More about this in the content creation post but it was very useful for everyone involved to get a sense of the direction of the demo.

![eon-storyboard](/assets/eon_storyboard.jpg)

The above drafts (latest version) is from 2017 and some things made it to the demo and some things didn't. Sometimes it's easy to look at a finished production and think that it was planned this way from the start. We had to try out lots of things to get where we actually ended up and in the end both from from a technical and design perspective.

## Next steps

The following weeks I will start describe all the parts of the demo. Some will will take longer than others so they will be split out over several posts depending on the complexity of each of them.
