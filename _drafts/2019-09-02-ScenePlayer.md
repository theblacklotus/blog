---
layout: post
title:  "The Tech Behind Eon: Scene Player"
author: Mikael Kalms (kalms/TBL)
author_link: https://twitter.com/kalmalyzer
---

This post describes the Amiga player for packed 3D scenes in our A500 demo Eon.

![woman2](/assets/woman2.png)

## Overview

We have [an offline packer for 3D scenes]({% post_url 2019-08-26-ScenePacker %}).
Just like the packer, the player required many iterations to perform well enough
to allow us to create the visuals we were after.

## Buffer usage

We have two versions of the scene-player; a 50fps 1-bitplane version and a
25fps 3-bitplane version. This text describes the 3-bitplane version.

We use four buffers for rendering:
1. Clear
2. Decode scene data + render lines
3. Blitter-fill
4. Display

The clear and blitter-fill are performed by a pair of interrupt-driven blits.
Line drawing is done using the CPU. The buffers are rotated every 2nd frame.

The clearing and blitter fill are straightforward; the interesting bits happen
during scene data decoding and line rendering.

## Scene data format

Each frame consists of a number of line strips. A line strip is N+1 points that form a sequence of N lines. Here is one such line strip:
```
    Num points in line strip: 18
      Full point: 0,179
      Full point: 0,90 - mask 0x1
      Delta point: 27,83 (delta 27,-7) - mask 0x1
      Delta point: 35,88 (delta 8,5) - mask 0x1
      Full point: 77,85 - mask 0x4
      Delta point: 94,76 (delta 17,-9) - mask 0x2
      Delta point: 93,82 (delta -1,6) - mask 0x2
      Delta point: 85,83 (delta -8,1) - mask 0x2
      Delta point: 97,101 (delta 12,18) - mask 0x5
      Delta point: 93,113 (delta -4,12) - mask 0x2
      Delta point: 102,121 (delta 9,8) - mask 0x2
      Delta point: 101,130 (delta -1,9) - mask 0x2
      Delta point: 83,127 (delta -18,-3) - mask 0x2
      Delta point: 52,133 (delta -31,6) - mask 0x2
      Delta point: 51,137 (delta -1,4) - mask 0x1
      Full point: 5,156 - mask 0x1
      Full point: 93,179 - mask 0x1
      Full point: 0,179 - mask 0x1
```

In order to reduce the data formats, consecutive points that are close to each
other are encoded using 3-bit X/Y deltas. Points that are further apart are stored as
full 16-bit positions. The mask value is the XOR of the colour on the left versus
the right side of the line.

## Decoding scene data

The scene data decoding needs to strike a balance between small on-disk footprint
and high decoding performance. In order to minimize the on-disk footprint, we
store most data as bit-streams rather than byte/word sequences. In order to maximize
decoding performance, we separate out data into different streams, so that each
stream only contains values of one specific bit-length.

Since each stream only contains values of one given bit length, we can
perform bulk decoding of tricky formats (in our case, 3-bit and 6-bit streams) before
the individual values are needed. Bulk decoding of those formats eliminates a lot
of shifting and conditional logic.

Here is an example of how we decode eight 3-bit values into 16-bit values:

```m68k
DECODE8		MACRO	fc_mask,temp0_cleared_upperbytes,temp1,temp2

		; fc_mask - contains $fc in lowest bytes
		; temp0_cleared_upperbytes - contains $000000 in top 3 bytes

		; Read 3 bytes, output 8 entries

		; a2    - input stream
		; a3+16 - lookup table for     a2a1a0b2b1b0---- -> word with a, word with b
		; a3    - lookup table for             c2c1c0-- -> word with c
		; a4    - lookup table for     d2d1d0e2e1e0---- -> word with d, word with e
		; a5    - lookup table for f1f0g2g1g0h2h1h0f2-- -> word with f
		; a6    - lookup table for f1f0g2g1g0h2h1h0---- -> word with g, word with h

		moveq	#0,\3
		move.b	(a2)+,\3		; Fetch %a2a1a0b2b1b0c2c1
		move.l	\3,\4
		and.b	\1,\3			; %a2a1a0b2b1b0----
		move.l	Lookup_AB-Lookup_C(a3,\3.l),(a1)+	; Lookup a,b
		eor.b	\3,\4
		move.b	(a2)+,\2		; Fetch %c0d2d1d0e2e1e0f2
		add.b	\2,\2			; %d2d1d0e2e1e0f2--
		addx.b	\4,\4			; %c2c1c0
		add.b	\4,\4			; %c2c1c0--
		move.w	(a3,\4.l),(a1)+		; Lookup c
		moveq	#0,\3
		move.b	(a2)+,\3		; Fetch %f1f0g2g1g0h2h1h0
		move.l	\2,\4			; %d2d1d0e2e1e0f2--
		and.b	\1,\2			; %d2d1d0e2e1e0----
		eor.b	\2,\4			; %f2--
		move.l	(a4,\2.l),(a1)+		; Lookup d,e
		add.w	\3,\3			; %f1f0g2g1g0h2h1h0--
		add.w	\3,\3			; %f1f0g2g1g0h2h1h0----
		or.w	\3,\4			; %f1f0g2g1g0h2h1h0f2--
		move.w	(a5,\4.l),(a1)+		; Lookup f
		move.l	(a6,\3.l),(a1)+		; Lookup g,h
		
		ENDM
```

At the end of the decoding we have a list of line segments with associated XOR values.
This list feeds directly into the line rendering.

## Rendering lines

Line rendering can be sped up by reducing the total number of lines processed,
reducing the per-line setup cost, and reducing the per-pixel overhead. We settled on
rendering lines to all affected bitplanes at the same time; it reduces both per-pixel
overhead and per-line setup cost.

This is the innerloop for a line that is to be rendered to bitplane 1 and 3:
```m68k
		REPT	179
		bchg	d2,(a0,d2.w)	; draw pixel in bpl1
		bchg	d2,(a2,d2.w)	; draw pixel in bpl3
		addx.l	d1,d0		; advance byte-step and step down one line
		addx.l	d3,d2		; advance bit-step
		ENDR
		bchg	d2,(a0,d2.w)	; draw pixel to bpl1
		bchg	d2,(a2,d2.w)	; draw pixel to bpl3
```

The line rendering uses two [DDA](https://en.wikipedia.org/wiki/Digital_differential_analyzer_(graphics_algorithm)) steppers - one for "current byte" and one for
"current bit within current byte". The dual DDAs remove the need for per-pixel
shifting and masking. It works as long as both DDAs have exactly the same precision
when stepping (so the bit-step DDA may need to have its lowermost bits masked out
during initialization).

## Back to the packer

Once we had decided on the rendering method, we added metrics to the packer: how many
cycles for per-pixel rendering to a single bitplane, how many cycles for per-pixel
rendering to two bitplanes, how many cycles for line setup etc. We then transformed
the data to minimize the metrics. Permuting the palette for each frame, in particular,
is done with the intention to minimize the total number of cycles spent in per-pixel
processing in the player.

## Discarded and untested methods

We have tried having pre-generated code segments for drawing all up-to-16-pixel
long segments. We scrapped that because the pre-generate code ate lots of precious
memory while only giving a minor speed boost. We have also tried using rather
inaccurate interval-halving when computing pixel locations for lines covering a small
number of lines vertically. We scrapped that as well because the reduced per-line
overhead was offset by the increased per-pixel overhead.

One thing that could have helped, is drawing lines with both CPU and blitter
(alternating according to some set of rules). We did not try that since it would be
quite complex to make it work without performance stalls and small pixel glitches. 
Instead, we chose to optimize our content further to make it all run at 25fps.