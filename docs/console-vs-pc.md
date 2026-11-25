# A console is not a PC

The Xbox Series X is close enough to a PC to be an x86-64 machine with an AMD
GPU, and different in exactly the places that decide how a renderer is
written: the memory model, the API surface, how shaders are compiled, and how
a finished frame reaches the display. This note is the picture the crates keep
pointing at. Numbers are Microsoft's published Series X specification.

## Hardware

| | Xbox Series X | A PC | Why it matters |
|---|---|---|---|
| CPU | 8 Zen 2 cores, 16 threads, 3.6 GHz with SMT and 3.8 GHz without, fixed | arbitrary cores and clocks, SMT optional | fixed clocks and core count let a title assume timing; the host does not have it |
| GPU | RDNA2, 52 CUs at 1.825 GHz, 12.15 TFLOPS FP32 | NVIDIA, AMD or Intel, different ISA and scheduling | shader behaviour is vendor-specific; a console binary is built against one known ISA |
| Memory | 16 GB GDDR6, unified with the CPU: 10 GB at 560 GB/s, 6 GB at 336 GB/s, one address space | discrete VRAM behind PCIe, or integrated UMA | a title assumes zero-copy, cache-coherent data the CPU wrote; on a PC that becomes upload heaps and explicit copies |
| Storage | 1 TB NVMe, 2.4 GB/s raw, about 4.8 GB/s compressed through a hardware decompression block | any SSD through OS file APIs | titles stream and decompress with hardware the host has to stand in for (DirectStorage is the PC echo) |
| Buses | CPU, GPU, memory controller and video encode on one die, no PCIe | PCIe links with real latency | synchronisation and cache-flush assumptions differ |
| Output | fixed modes, HDR10, 4K120, VRR, fixed refresh | windowed or borderless, compositor in the middle | presentation semantics differ |

## Rendering

1. **The API is D3D12.X, a superset of D3D12.** `ID3D12DeviceX`, command lists
   with extra states, permissive state combinations, explicit aliasing and
   placed resources, typed UAV loads, and console-only calls (`XMemAlloc`,
   the `XG*` family). PC D3D12 is smaller and more opinionated, so the HLE has
   to synthesise the missing permissiveness instead of passing calls through.

2. **Memory is explicit and unified.** A title places resources at addresses it
   chooses, aliases them, and writes descriptor memory itself. PC D3D12 hands
   out opaque driver-created descriptor heaps, so the title's own descriptor
   writes have to be shadowed and mirrored into host heaps.

3. **Shaders arrive precompiled for a known GPU.** Console blobs are DXBC/DXIL
   with Xbox extensions, already bound to RDNA2 semantics (wave32/64, quad
   operations, barycentrics, derivative behaviour). No PC driver accepts them:
   they are translated to DXIL and then compiled by the vendor driver. Two
   compilers, two error surfaces, and a translate-then-compile cost per
   pipeline.

4. **The console has a compositor plane; the PC has a swap chain.** A title may
   present a plane the compositor fills. On the PC that plane is a resource
   nobody wrote, and the window shows whatever the swap chain points at.

5. **Compression metadata is invisible on the console.** Depth and colour
   surfaces carry DCC and HTILE metadata the hardware manages. On the PC that
   is a vendor extension the emulator either implements or bypasses; see
   [texture metadata](texture-metadata.md).

6. **Barriers are stricter on the PC.** Console states are broad and forgiving;
   PC D3D12 wants promotion, demotion or explicit transitions, or the driver
   silently drops work or faults. This is how a title renders correctly on the
   console and draws nothing on a PC.

7. **Submission model.** The console has dedicated hardware queues and a
   command processor; the PC exposes three queue types that drivers schedule
   differently. Overlap the console gets for free can serialise on the host.

8. **The OS and module set differ completely.** Xbox has its own kernel, XAPI,
   XAudio2 variants and Xbox Live modules, with no Win32 and a different
   exception and page-fault model. That is the HLE layer.

## Where this lands in this tree

| difference | crate | the shape it takes |
|---|---|---|
| OS, kernel, modules | `victory-hle` | implemented system modules; nothing is passed through to Windows |
| API surface, descriptors, barriers | `victory-gpu` | host heap mirroring, state remapping, the swap chain |
| shader binaries | `victory-shader` | console blob to DXIL, then the vendor driver compiles it |
| image and texture formats, metadata | `victory-formats` | tiling, HTILE and DCC, format conversion |
| streaming, storage | `victory-loader`, `victory-pe` | file systems and the decompression path |

## What it costs, concretely

- The rendering maths can be right and the hand-off wrong: the compositor
  plane (point 4), which is the class behind a window that stays black while
  finished art sits in a render target.
- Shaders must be translated, not loaded, so a draw can be dropped because a
  pipeline could not be built (points 3 and 6).
- Descriptors and memory must be shadowed, so their lifetime is ours to get
  right (point 2).
- VRAM is not unified: 16 GB of console memory becomes a budget on a discrete
  card, and a pipeline the driver refuses for want of memory is a draw that
  never happens.

See also [architecture](architecture.md) for the crate layout, and
[debugging notes from a port](debugging-notes-from-a-port.md) for the failure
shapes this produces.
