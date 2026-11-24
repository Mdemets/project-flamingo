# Texture metadata planes

Render targets and depth buffers on the console carry compression metadata
after their data: HTILE for depth, DCC for colour, CMASK and FMASK for
multisampled colour. The console's layout library (`Gfx10Lib`) computes the
planes' size, alignment, pitch and height together with the data layout, and
titles read those numbers back from the layout they asked for. Grand Theft
Auto VI also keeps texture sizes in its own tables and asserts when the
layout it receives disagrees with them.

`victory-formats::texture` computes the data planes only (mips, slices,
tiling, detiling). `Texture::with_metadata` adds the HTILE and DCC planes
for a chip whose `MetaConfig` the caller knows; `Texture::new` stays
metadata-free, because the values are not known for either console revision
yet and a wrong one changes every render target's size.

## What is ported

From `gfx10addrlib.cpp`, for a thin 2D resource with a block size of one
element (no block compression):

| addrlib | here | Gives |
|---------|------|-------|
| `GetMetaBlkSize` | `meta_block` | the metadata block's size and the data block it covers |
| `GetMetaOverlapLog2` | `meta_overlap` | the data a metadata block overlaps |
| `GetEffectiveNumPipes` | `effective_pipes` | the pipe count after `rbPlus` |
| `GetPipeRotateAmount` | `pipe_rotate` | the pipe rotation of an XOR mode |
| `GetBlk256SizeLog2` | `micro_block` | texels in the 256-byte micro block |
| `HwlComputeHtileInfo` | `htile` | HTILE pitch, height, size, alignment |
| `HwlComputeDccInfo` | `dcc` | DCC pitch, height, size, alignment |

Each plane is placed after the planes already in the texture, aligned to its
own alignment; the texture's `size` covers it and its `alignment` is the
larger of the data and metadata alignments. That is the console's
`HwlComputeMeta2DInfo` placement.

What a description must be for a plane to be added:

* HTILE: 2D, one mip, one sample, 32-bit element, `SW_64KB_Z_X`, block of
  one element.
* DCC: 2D, one mip, an `_R` or `_R_X` swizzle, block of one element. Depth
  formats get none, as on the console.

Anything else gets no metadata plane: a wrong plane would be worse than a
missing one, because the title reads the numbers.

## The configuration

`MetaConfig` holds what addrlib reads from GPU registers rather than from a
texture description:

| Field | addrlib | Series X/S value |
|-------|---------|------------------|
| `pipes_log2` | `m_pipesLog2` | not known |
| `pipe_interleave_log2` | `m_pipeInterleaveLog2` | not known |
| `num_sa_log2` | `m_numSaLog2` | not known |
| `max_comp_frag_log2` | `m_maxCompFragLog2` | not known |
| `rb_plus` | `supportRbPlus` | not known |

Series X and Series S do not have to share one, so guessing is not an
option: a wrong block size makes every render target disagree with the
console's tables. `MetaConfig::parse` reads the string form
(`pipes=2,interleave=8,sa=1,maxfrag=3,rb+=1`), so a candidate can be tried
without a rebuild; `MetaConfig::to_string` prints it back for logs.

## Calibrating

The values can be found from one console-truth size: allocate a texture
whose description carries a plane, compare the size the title expects with
the one the layout gives, and repeat for the candidates until they agree.
The space is small (pipes 1..=4, interleave 0..=12, sa 0..=2, maxfrag 0..=4,
`rb_plus` 0 or 1: 1,560 configurations), and one probe of each kind fixes
most of it: a DCC plane exercises `maxfrag` and the colour block size, an
HTILE plane the depth one.

A candidate that makes the world render differently, rather than asserting,
is still unproven: the planes are metadata for compression the host GPU does
not use, so only their size and placement can be observed from here.

## Not done

* CMASK and FMASK (multisampled colour): their sizing goes through the same
  `GetMetaBlkSize` but with arguments this port does not pin down yet.
* Mip chains: addrlib places the metadata of a mip chain over the base
  level with its own tail rules; `with_metadata` adds nothing for
  `mips > 1`.
* `Tex1d`, `Tex3d`, block-compressed formats: no metadata.
* Reporting the planes to the title: `victory-hle`'s `xg` tells the title
  about data planes only (`PlaneOut::usage` 1, 7 and 8). Whether the console
  reports metadata planes there, and with which usage codes, is not known.

## Wiring it in

Both layout call sites construct `Texture::new`: `victory-gpu`'s
`texture_layout` (`resource.rs`, which feeds `footprint` and the host
allocation) and `victory-hle`'s `texture` (`xg.rs`, the layout the title
reads). To turn metadata on for an experiment, build the description first
and call `Texture::with_metadata(&desc, &config, request)` when a
configuration is set, leaving `Texture::new` otherwise. Keep it off by
default until the values are verified against the console's tables.
