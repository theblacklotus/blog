---
layout: post
title:  "The Tech Behind Eon: Track Loader"
author: Andreas Fredriksson (dep/TBL)
author_link: https://twitter.com/deplinenoise
---

This post describes the track loading infrastructure that drives Eon.

## Bootblock

When the demo starts from a floppy, the OS will automatically load the first
1,024 bytes into memory and jump there with some predefined registers set up.
This is the boot block.

The Eon bootblock is pretty full. It:

- Locates the topaz font from memory (used for the crash screen)
- Disables multitasking and enters supervisor state
- Disables caches
- Loads the second stage Loader into some scratch memory using the normal `DoIO()` mechanism
- Moves the rest of the bootblock to chip RAM (as some kickstarts will load it into fast mem)
- Relocates the stack to chip mem
- Probes for a workable memory configuration. From floppy we support 1MB chip, or 512 k chip and any amount of fast memory greater than 512k (or slow mem).
- If no workable memory configuration is found, shows a "Fastmem needed" screen, with a small copper list.
- Disables the OS
- LZ4 decompresses the loader into it's final place at the base of fast memory (or second half of chip memory)
- Patches up relocations for the loader to handle any memory config
- Clears BSS for the loader
- Sets up two parallel stacks; one for the parts and one for the loader. These are at the base of fast memory.
- Transfers control to the loader

Most of this is relatively straight forward, but it was challenging to fit it all in there. LZ4 compression was a big deal for disk space as it saved a few kb, and we were able to add the decompressor in under 100 bytes of code. It's very slow obviously, but that doesn't matter in this case.

## Loader

Now the second stage loader takes over. This is a much larger program and when
it is started all the memory has been reserved for the demo. We reserve the
very bottom of chipmem for the VBR, and the first 8k of fast memory for stacks,
but the rest is set aside into two big slabs of memory.

The loader is written in a mix of assembly and C. All the high-level sequencing
is C code, which makes it easy to iterate on and debug. The heavy lifting, such
as MFM decoding and LZ4 decompression is done in assembly code.

The loader has quite a few jobs:

- Manage memory allocation (see below)
- Load, decompress, reloc patch and initialize the next part before it needs to start
- Handle the disk controller machinery
- Handle part switching
- Play music (includes P61)
- Stream and decompress samples
- Mix down drum loops as needed

### Threading

We handle the above tasks in the "background" when the current part is
inactive. Parts can span more than one frame (for example the 3D scenes run at
25 Hz). Whenever a part is done updating, it yields the CPU by triggering a
`SOFTINT` interrupt. The interrupt handler for this interrupt swaps stacks and
returns into the loader thread, which happily continues doing whatever it was
doing previously.

Each vblank we swap back to the current part, so it gets a new update in a
predictable time frame. The basic idea is that any CPU time the parts are not
using will go to the loader machinery automatically, and parts can still use
both a vblank routine and a main update routine.

Overall this scheme worked great, but we had a few false starts trying to use
CIA timers to guarantee the loader thread can make progress. We opted for the
described cooperative model instead, and we simply check that the loader can
make some progress now and then. We had issues where if a part would take
almost exactly 1 frame the loader would get just a couple of hundred cycles and
would take forever to load the next part, where as on a slower machine it would
get almost a full frame because the effect ran at 25 Hz.

In the end predictability is what matters the most. We framelock everything and
time everything so we never drop frames on A500, and then we just waste time on
faster CPUs. This means that the loader can always make progress on all
machines.

One neat feature about this is that we can run arbitrary initialization code
in each part in the "background" while the previous part is still rendering.
This means that it's OK to compute lookup tables on the fly and things like
this without causing any slowdowns on part switching.

Parts get this set of callbacks:

- Init (called on loader thread, can run for abitrary time)
- First frame (called on vblank interrupt on first frame of rendering)
- Vblank (called on vblank interrupt while part is active)
- Last frame (called on vblank interrupt just before switching part out)
- Update (called on demo main thread, can run for more than one frame)

The loader maintains a small API surface for parts to call out into the
loader do things like get the address of the shared render buffers. It
is a simple array of function pointers that is passed into each part's
init callback.

## Memory Allocation

We partition memory in a simple ping-pong scheme. There is no dynamic memory
allocation anywhere in the demo. Instead each part is fully statically linked
with all the data it needs, as well as work buffers in the form of BSS data.
This works really well, as we can determine offline if the demo will fit in
memory without really running it.

This means that part N and part N+1 have to fit in memory at once. We allocate
alternatively from the bottom and top of the chip/fast block, and we don't need
any error handling because we've statically determined everything will fit.
(Although we had some fun times when the offline calculations were broken.)

In fact the memory allocator is so simple I can paste it here:

    static void* ldr_alloc_part(MemRange* r, size_t size, int direction)
    {
      // Round up to 16 bytes because our checksummer needs that.
      size = ALIGN_ARB(size, 16);

      if (direction)
      {
        char* point = r->m_AllocPoint - size;
        r->m_AllocPoint = point;
        return point;
      }
      else
      {
        char* point = r->m_AllocPoint;
        r->m_AllocPoint += size;
        return point;
      }
    }


Our debugging configuration needs more space as it packs in a bunch of text
data and printfs, so we simply discard some graphics in that version like
subpixel scroll images to claw back some space.

The memory map for the demo looks something like this:

Chip block:

- VBR
- Track Buffers (2 independent buffers for floppy DMA)
- Shared screen space (big block of render target data all parts can use)
- Music chip section (space for samples - around 200K)
- Dummy copper list
- Rest reserved for parts (split in two parts in a ping-pong scheme)

Fast block (actually in high CHIP if machine only has chipmem):

- Loader stack (4K)
- Demo stack (4K)
- Loader executable + buffers
- Small shared state for part communication (used for special handover stuff)
- Music fast section (patterns)
- Rest reserved for parts (split in two parts in a ping-pong scheme)

## Floppy Controller

The floppy control machinery juggles floppy DMA. There is a simple request
mechanism where the main loader code requests data based on a static table
bundled on to the floppy by the layout tool. The table is an array of
`DiskRange` structs:

        typedef struct DiskRange
        {
          uint32_t  m_DiskOffset;
          uint32_t  m_MemSizeBytes;
          uint32_t  m_UninitializedSizeBytes;
          uint32_t  m_CompressedDiskSizeBytes;
          uint32_t  m_UncompressedSizeBytes;
          uint32_t  m_CompressedChecksum;
          uint32_t  m_UncompressedChecksum;
          uint16_t  m_SafetyMargin;
          uint16_t  m_CompressionType;          // kCompressionType*
        } DiskRange;

        COMPILE_ASSERT(sizeof(DiskRange) == 32);

Given these the high-level loader code calls into the floppy controller to pull
ranges in as it goes. The floppy controller always attempts to have the _next_
track in flight, so that I/O is always overlapped with decompression.

## LZ4 decompression

We use LZ4 compression for all lossless data. We tried other compressors but we
didn't have enough CPU budget to go with anything more complex. The decompressor
is 8-way unrolled assembly and gets great decompression speed compared to something
like zlib.

There are two quirks with the decompression. First of all, we needed to to decompress
in place as we can't fit the compressed data and the decompression buffer in memory
all at once. We handle this by loading the compressed data at the end of the
decompression buffer and then run the decompression so that it chases the tail of the
buffer. LZ4 is a simple format, so we can compute the safety margin needed to do this
safely.

The other quirk is more complicated. In order to overlap decompression and I/O we
need to be able to interrupt and resume the decompression. Normally LZ4 decompresses
the whole thing in one go, which wasn't suitable for us. So what we do is compute
a number of _safe points_ where the decompression can pause without any ill effects.
We store these as a compact table at the start of the compressed data, so we can drive
the decompressor in smaller pieces.

First, we cap the maximum number of chunks to 16. Then, we limit each chunk to 8K.
Now, looking at the size of a piece of data in the table we can do a simple log2
computation to see how many safe points must follow. Next we enter a loop where
we mix I/O requests with decompressing each safe point.

The offline logic to find the safe points looks like this:

    while (p != end)
    {
      update_max();

      // Record the current position as a safe resume point if we've advanced enough in the compressed stream.
      // Note that this also filters out the start of the stream (0) as that is implicitly safe.
      {
        const uint32_t offset = uint32_t(p - lz4_data);

        // Be careful that we only emit 2-byte aligned safe points, to avoid messing up MFM decoding loops at runtime
        if (((offset | output_count) & 1) == 0 && offset - prev_safe_point >= LDR_COMPRESSED_CHUNK_SIZE)
        {
          output.push_back(offset);
          prev_safe_point = offset;
        }
      }

      const uint8_t b = *p++;
      update_max();

      // Decode token into literal and match lenghts
      uint32_t literal_length = b >> 4;

      // Decode additonal literal length bytes (if any).
      if (0xf == literal_length)
      {
        uint8_t extra_byte;
        do
        {
          extra_byte = *p++;
          update_max();
          literal_length += extra_byte;
        } while (0xff == extra_byte);
      }

      // Skip the literal.
      p += literal_length;
      output_count += literal_length;
      update_max();

      if (p == end)
        break;

      // Skip match offset
      p += 2;
      update_max();

      uint32_t match_length = b & 0xf;

      // Minimal match length is 4
      output_count += 4;
      update_max();

      if (0xf == match_length)
      {
        uint8_t extra_byte;
        do
        {
          extra_byte = *p++;
          update_max();
          match_length += extra_byte;
        } while (0xff == extra_byte);
      }

      output_count += match_length;
      update_max();
    }

What this is doing is essentially walking the LZ4 format and figuring out where safe breaks can be placed.

## Layout tooling

All parts are built in two flavors:

- Standalone test version: this is a regular executable that links with a dummy startup system that pretends this is the only part in the demo. Useful for iterating on a single part.
- A floppy version: this is a stripped HUNK executable that relies on the demo framework in the loader to work

This is all handled by some custom build rules in our Tundra setup. Here is what a part definition looks like:

    Part {
        Name = "PillarOfLight_I",
        Depends = { "util", "blitter", "copper", "sprite", "alembic_amiga", "palette_flicker" },
        Sources = {
            "Projects/Pandora/PillarOfLight_I/PillarOfLight_I.c",
            "Projects/Pandora/PillarOfLight_I/PillarOfLight_I.d68",
            ScanIncBins { Source = "Projects/Pandora/PillarOfLight_I/PillarOfLight_I.c" },
        },
        Config = { "amiga" }
    }

We then collect a master table of the parts in another Lua file that can be
consumed both by the build system (to manage build dependencies) and by the
layout tool. The file looks like this:

    BeginFloppy("disk1.adf")
    Bootblock("Bootblock")
    Loader("Loader")
    Music("Data/Music/P61.sketch-45", "Data/Music/SMP.sketch-45", "Data/Music/samples.bin", 206568)
    Part("TBLLogo", 0)
    Part("MobileIntro", PatternToFrameIndex(4))
    Part("SpaceIntro", PatternToFrameIndex(7) - 100)
    Part("SolarEclipse", PatternToFrameIndex(10))
    Part("SpaceDescend", PatternToFrameIndex(14))
    Part("ForestLines", PatternToFrameIndex(18))
    Part("PillarOfLight_I", PatternToFrameIndex(20))
    Part("PillarOfLight_II", PatternToFrameIndex(24))
    Part("SpaceAscend", PatternToFrameIndex(28))
    Part("ParticleSphere", PatternToFrameIndex(30))
    Part("AlleyScenePart1", PatternToFrameIndex(34))
    Part("AlleyScenePart3", PatternToFrameIndex(38))
    EndFloppy()

    BeginFloppy("disk2.adf")
    RawString("TBL-2019")
    Part("TexturedRubberCube", PatternToFrameIndex(46))
    Part("TransitionEffect0", PatternToFrameIndex(50))
    Part("WomanFlatWideShot", PatternToFrameIndex(52))
    Part("WomanFlatShade", PatternToFrameIndex(56))
    Part("WomanFlatWideShot2", PatternToFrameIndex(64))
    Part("WomanFlatShadePart3", PatternToFrameIndex(68) - 4)
    Part("WomanFlatShadeFloat", PatternToFrameIndex(78))
    Part("WomanFlatShadeFloat2", PatternToFrameIndex(80))
    Part("FloatingInSpace", PatternToFrameIndex(82))
    Part("SolarDeclipse", PatternToFrameIndex(84))
    Part("EndLogoCredits", PatternToFrameIndex(86))
    EndFloppy()

The build system and the layout tool can source this file with different
definitions for the functions used (like `Part` and `Loader`) to do the
appropriate thing. For example, the build system simple collects what
parts are used, where as the layout tool records all the data in tables.

This system is neat, because touching a single shared source file will
rebuild all configurations and variants needed, all in parallel with full
dependency checking. Here is a sample session just touching a source file:

    ~/repo/newage_a500 (master) > touch Projects/Pandora/SpaceIntro/SpaceIntro.d68
    ~/repo/newage_a500 (master) > tundra2 amiga-mac-debug
    Deluxe68 t2-output/amiga-mac-debug-default/__SpaceIntro/SpaceIntro-d68-7835eb2d84a80c5cf29f126527b3f236.s
    Asm t2-output/amiga-mac-debug-default/__SpaceIntro/SpaceIntro-d68-7835eb2d84a80c5cf29f126527b3f236-s-16d9d6529e7fd026871b4d8a5c06ba8e.o
    Program t2-output/amiga-mac-debug-default/SpaceIntroTest
    Program t2-output/amiga-mac-debug-default/SpaceIntroFloppyVersion
    StandaloneFloppyLayout t2-output/amiga-mac-debug-default/SpaceIntro.adf t2-output/amiga-mac-debug-default/SpaceIntro.txt
    Program t2-output/amiga-mac-debug-default/SpaceIntroTestMusic
    FloppyLayout Projects/Pandora/FloppySpec.lua
    disk1.adf: used size: 870180 bytes (30940 bytes free)
    disk2.adf: used size: 886300 bytes (14820 bytes free)
    CreateEmbeddingAsm t2-output/amiga-mac-debug-default/embedded-disk2.adf
    CreateEmbeddingAsm t2-output/amiga-mac-debug-default/embedded-disk1.adf
    FloppyLayout Projects/Pandora/FloppySpec.lua
    disk1.adf: used size: 887682 bytes (13438 bytes free)
    disk2.adf: used size: 886300 bytes (14820 bytes free)
    Asm t2-output/amiga-mac-debug-default/__InMemoryLoader/_embed_disk1-s-cab0c383b465a0c030ad8a1dd50f3952.o
    Asm t2-output/amiga-mac-debug-default/__InMemoryLoader/_embed_disk0-s-cab0c383b465a0c030ad8a1d324ab482.o
    Program t2-output/amiga-mac-debug-default/InMemoryLoader
    *** build success (3.05 seconds)

This takes just 3 seconds on an old macbook pro, so it's pretty speedy.

The layout tool itself does a bunch of work:

- Loads all executables and strips them; fixing up relocations and references as needed. It is essentially a full HUNK linker.
- It strips dead code (as we link with static libraries this comes in handy)
- Merges sections together so we end up with three partitions:
  - One FAST section (payload + bss + uninitialized bss)
  - One CHIP section (payload + bss + uninitialized bss)
  - One relocation section
- Statically checks memory allocation given the above data
- Compresses all data and samples and locates safe points
- Patches loader and bootblock with final streaming tables
- Writes debugging data in the form of symbols and labels for the various tools we have for the emulator
- Finally writes ADF images

Relocations are compressed using a scheme to minimize size and maximize
patching speed. We sort all relocation offsets, and then emit relocations that
lie within 32k of the previous relocation point using 2 bytes. If the high bit
is set, it means we must pull in two additional bytes and make a larger jump.

Each such run of offsets to patch is preceeded by a count, where the two low
bits encode the source and destination segments. Because we only have one CHIP
section and one FAST section this is enough, and saves data for each relocation.
This data is then LZ4 compressed as well.

The runtime relocation patcher looks like this (in d68 form):

            @cproc ldr_patch_relocs(a0:chip_base,a1:fast_base,a2:relocs,d0:relocs_size)

            @areg	relocs_end
            lea.l	(@relocs,@relocs_size.l),@relocs_end
            @kill	relocs_size

            Printf	"Patching relocs relocs=%p, chip_base=%p, fast_base=%p, relocs_end=%p",@relocs,@chip_base,@fast_base,@relocs_end

    .control_loop	cmp.l	@relocs,@relocs_end
            beq.s	.done

            @dreg	ctrl

    .section
            ;Printf	"Pulling from relocs=%p",@relocs
            move.w	(@relocs)+,@ctrl		; pull control word

            ; Use lower two bits to figure out src/dst
            @dreg	adjust
            @areg	patch_seg

            @dreg	mask
            move.w	#$7fff,@mask

            move.l	@chip_base,@adjust
            lsr.w	#1,@ctrl
            bcc.s	.dst_ok
            move.l	@fast_base,@adjust
    .dst_ok
            move.l	@chip_base,@patch_seg
            lsr.w	#1,@ctrl
            bcc.s	.src_ok
            move.l	@fast_base,@patch_seg
    .src_ok
            @rename	ctrl count
            ;Printf	"Patching relocs in seg %p, predec count=%d",@patch_seg,@count
    .patch_loop
            @dreg	d
            moveq.l	#0,@d
            move.w	(@relocs)+,@d
            bpl.s	.patch

    .long	and.w	@mask,@d
            swap	@d
            move.w	(@relocs)+,@d

    .patch
            add.l	@d,@patch_seg
            ;Printf	"Adding %p at %p, delta=%ld",@adjust,@patch_seg,@d
            add.l	@adjust,(@patch_seg)
            dbf.s	@count,.patch_loop
            @kill	count,adjust,patch_seg

            bra.s	.control_loop

    .done
            @endproc


## Standalone version

For the standalone (workbench) version of the demo, we started off with a
completely custom build that used all kinds of macros to glue stuff together.
However it was flaky and brittle, and we wanted something guaranteed to work.

It struck us that we can simply turn the second stage loader into a system
friendly startup program, and then embed the two disk images in memory. Instead
of disk DMA, we'd just issue a memcpy. There is a little more to it than that,
but in a nutshell that's how it works.

Because of our super simple memory allocation scheme, we can just set aside
two 512 KB buffers and initialize the loader slightly differently and then the demo
will run the same way as from floppy (minus the disk swapping, of course).

