---
layout: post
title:  "The Tech Behind Eon: Subpixelled Blitter lines"
author: Mikael Kalms (kalms/TBL)
author_link: https://twitter.com/kalmalyzer
---

One piece of tech that we implemented for the demo, but hardly made use of, is blitter lines with subpixel precision.

The only place in the demo where it is used here:

![solar-eclipse](/assets/solar-eclipse.png)

It may not be immediately obvious, but the sun includes a rotating white disc.
It is made up of 40 line segments, where the vertices rotate and animate slightly according to a noise function.
The result would have been jittery without subpixel precision. 3 bits of subpixel precision is enough to make the result smooth.

For an in-depth review of how the subpixelled blitter line implementation works, see [the source code repository](https://github.com/Kalmalyzer/subpixel-blitter-line).
The repository describes the algorithm, and contains a reference implementation.

Why didn't we use this tech in more places than this single part? In some places, we chose not to use it, because CPU constraints
(slower line setup) would force us to use fewer lines. In other places, we chose not to use it, because disk space constraints
(more bits to store per preprocessed vertex) would force us to use fewer lines. In one case, we wanted to use it, but
we were running into really strange bugs; we were not willing to invest the time to figure out what was going wrong,
so we went with a non-subpixelled line drawer instead.