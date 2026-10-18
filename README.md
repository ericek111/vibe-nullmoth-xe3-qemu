# A GPU-accelerated macOS VM on an Intel Panther Lake SR-IOV VF

macOS 26 Tahoe running in QEMU/KVM on a virtual function of the Panther Lake iGPU (Arc B390, Xe3).
[Nullmoth](https://github.com/nullmoth/nvidia-macos-driver)'s port of Linux's `xe` driver (NMXGPU) drives
the VF. Apps get Metal, OpenGL and Vulkan on the VF. WindowServer composites the desktop on it, and the
result is scanned out zero-copy to the QEMU window at the host's native resolution and 120 Hz.

This repository has the write-up and the patches: 47 to the Nullmoth driver tree, 50 to its xkpi
(LinuxKPI-on-xnu) layer, and small series for drm-kmod, libdrm, Mesa and QEMU. Nullmoth's README says
NMXGPU "has not been loaded on hardware"; here it drives a real (virtual) GPU.

## Results

| | |
|---|---|
| Desktop | GPU-composited by WindowServer on the VF; 2880x1710 @ 120 Hz, 1:1 with the host window; follows window resizes |
| Window drag | ~118 presents/s, settled (77 before the display work) |
| WindowServer, idle login window | ~0.1% CPU idle, 40-50% while animating (105-130% on the CPU-composited path) |
| Safari `requestAnimationFrame` | 60 fps, 101 of 101 frames exactly two refreshes apart on the 120 Hz display |
| Metal compute | 1M-element kernel exact; xe exec round trip 0.4 ms (Linux on the sibling VF: 0.3 ms) |
| OpenGL (Apple's GL-over-Metal renderer) | glbench 2000 draws/frame: 25.2 → 5.7 ms |
| Minecraft 26.3 (vanilla, render distance 16) | 32 fps (1-2 fps before the GL fixes below) |
| Sandboxed apps (App Sandbox, WebKit GPU process) | get the GPU |

## Setup

- **Host.**
  - Arch Linux (CachyOS), kernel 7.2.9.
  - Intel Panther Lake, Arc B390 iGPU `8086:b080` on `xe`, PF GuC 70.72.1.
  - Two VFs (`sriov_totalvfs` = 2). One is passed to the macOS VM with VFIO.
- **QEMU.** 11.1.2 plus the four patches in `patches/qemu`.
  - Machine: q35, OVMF without Secure Boot, hugepages, host-passthrough CPU.
  - virtio-vga with `blob='on'`, shown with GTK and `gl=on`.
- **Guest.**
  - macOS 26.7.1 (25G241), SMBIOS iMac20,1.
  - Booted by OpenCore 1.0.8 with Lilu, VirtualSMC and RestrictEvents.
- **Nullmoth.**
  - nvidia-macos-driver 2.0.2 (`d96eaca`).
  - Its xkpi tree builds NMXGPU from Linux 6.14 `xe` and FreeBSD's drm-kmod, on Nullmoth's BSD-on-xnu layer.
- **Userspace.**
  - NMXAccel: Nullmoth's NVIDIA accelerator kext, built as `NMX_ACCEL` for Intel/AMD.
  - Nullmoth's Metal driver bundle (NVMTL), with Mesa's ANV underneath.
  - Nullmoth's AIR → SPIR-V translator.
  - libdrm and Mesa with Nullmoth's macOS patches plus mine.

## Background: what Nullmoth ships, and why it didn't work on a VF

- **NMXGPU is Linux 6.14's `xe` compiled for xnu.**
  - It runs on FreeBSD's LinuxKPI, drm-kmod's DRM core and TTM, and Nullmoth's own BSD-on-xnu layer.
  - `fork_xe.py` makes about 14 anchored edits to Intel's sources.
  - `override/xe` holds about 1,200 lines (the userptr/HMM layer, the page-table walk, glue).
  - The rest of xe is unchanged.
- **NMXAccel and the Metal plugin.**
  - NMXAccel is Nullmoth's NVIDIA accelerator built with `-DNMX_ACCEL`.
  - The Metal plugin is their NVK-based one with ANV in its place.
- **On Intel and AMD, Nullmoth runs "headless".**
  - Apps render on the GPU and their frames are copied into the IOSurfaces WindowServer reads.
  - WindowServer itself composites on the CPU into a plain IOFramebuffer.
  - The GPU-composited display path (display machine, display pipes, AGDC) exists only for NVIDIA.

On a VF, that stack fails before it starts:

1. **The VF isn't recognised.** In 6.14 no platform sets `.has_sriov`. `ptl_desc` is only
   `{PANTHERLAKE, .has_display, .require_force_probe}`. xe never enters VF mode. It treats the VF as a whole GPU,
   loads firmware and touches PF-only registers.
2. **Several port-wide bugs.** These would have stopped NMXGPU on any hardware (next section).
3. **A VF has no display engine.** It also has no legacy interrupt line, no aperture BAR and no stolen memory.

## Part 1: xe on the VF

The VF-specific part turned out to be small. 6.14 already turns display, GSC, GuC power control and pcode off on a
VF, and the VF↔PF relay ABI matches the 7.2 host (relay 1.0, VF ABI 1.2). The one missing gate was
`.has_sriov = true` in `ptl_desc`. Everything after that was bugs in the port:

- **xe never attached to any GPU.** LinuxKPI registers a driver on `vgapci`, where NMXGPU offers display devices,
  only under the name `drmn`. xe registered as `xe` and so attached to nothing. (xkpi `dbe3258`)
- **Every GuC reply was dropped.** LinuxKPI's `pm_runtime_get_if_active/in_use` are `-EINVAL` stubs, so xe's G2H
  worker and fast path never ran. (`7b5628f`)
- **Kernel mappings of non-contiguous BOs had holes.** A multi-range `kIOMemoryTypePhysical64` descriptor maps only
  its first run. Pages are now mapped with one descriptor per physical run through `IOMultiMemoryDescriptor`. This
  showed up as a panic in `xe_lrc_init`'s memcpy. (`74b1de6`)
- **6.14 VF bugs and ordering.**
  - The root GT's MMIO must be set up before the early GuC `GMD_ID` query: a NULL MMIO base panic. (`0b7902a`)
  - CT replies are polled while IRQs aren't installed yet, because 6.14 handshakes with the PF before
    `xe_irq_install()`. (`e190cd4`)
- **Interrupts.** IOPCIFamily publishes only the legacy line until `configureInterrupts()`, and a VF has no INTx.
  NMXGPU now asks for an MSI vector. (`31ccc70`)
- **Power.** IOKit's initial "power on" was forwarded as `device_resume`. That restarted the running GPU, sent the
  GuC submission count to -1, and created every user exec queue paused. Resume now happens only after a suspend.
  (`1d7e415`)
- **Shutdown.** The VF skips the driver FLR (PF-only registers) and resets through `VF2GUC_VF_RESET`, backported
  from upstream. Before, every reboot waited out a 3 s timeout. (`3251fbf`)
- **Backports from Linux 7.2.** Each was tested separately. (`8ee357f..a4fb78b`)
  - Memory-based interrupt dispatch handles user and GuC interrupts every time, as Intel's programming note asks.
  - Three Xe3 per-context workarounds: 14026781792, 22021007897 and 14024681466.
  - `READ_ONCE`/`WRITE_ONCE` on the G2H fence done flag.
  - `xe_rpm_reclaim_safe` without `!has_sriov`. This also ended a flood of `drm_WARN_ON(handle_system_ccs)`.

Why not move to a 7.x `xe` outright? xe itself is barely modified. What pins it to 6.14 is drm-kmod, whose
upstream stops at Linux 6.14. A 7.x xe needs DRM core and TTM pieces drm-kmod doesn't have. Backporting a handful
of VF-relevant fixes was the cheaper step.

## Part 2: Metal on the VF

- **/dev/dri.**
  - Opens failed until each open got a vnode and `d_fdopen`'s ENXIO-after-finit convention was accepted.
  - libdrm identifies clone nodes by name, because `stat` reports the DRM minor.
  - `dev_is_pci` was false for DRM children, so `GET_PCIINFO` failed.
  - Every `stat` leaked a clone minor.
  - (xkpi `78ad1f8`, `571826f`, `edceb3b`; libdrm 0001.)
- **BO mapping.**
  - `nmx_mmap` checked alignment with LinuxKPI's `PAGE_MASK` and rejected every offset. (`7658025`)
  - ANV maps BOs through `NMX_DRM_IOCTL_MMAP` on Darwin. (Mesa 0001)
- **Wait timeouts.** These are reported as macOS `ETIME` (101). Mesa treated the BSD value 60 as device lost.
  (`189ddf5`)
- **Page tables.** The xe page-table walk gives callbacks a child copy, and `post_descend` runs only after a real
  descent. A stale child faulted in `VM_BIND`. (`5d3a810`)
- **Interrupt bookkeeping.** Critical sections and spin mutexes now nest per thread. Per-CPU counts broke when a
  thread blocked inside one, and returned to user space with interrupts off. (`709153a`, `98a0b22`)
- **Cache coherency.** Two bugs, both producing wrong output in CoreAnimation:
  - xnu maps RAM write-back even when a user mapping asks for write-combining, so CPU uploads sat in the CPU cache.
    ANV on Darwin iGPUs now uses the cached-coherent PAT for everything. (Mesa 0004)
  - TTM clears pooled WC pages through the write-back kernel view, and the dirty lines later overwrote GPU writes.
    `kunmap` and `clear_page` now write non-WB pages out of the cache. (xkpi `424ff26`, `10bddab`)
- **Compression.** Xe2+ compression is off by default on Darwin, opt in with `ANV_XE2_COMPRESS=1`. VF CCS needs
  7.2-era kernel work.

## Part 3: sandboxes

macOS sandboxes cut most GPU clients off from `/dev/dri`:

- **WindowServer's sandbox denies ioctls on device files.**
  - NMXGPU publishes an IOKit user client, `IOAccelNMXDRMClient`, type `NMXD`, that carries DRM ioctls.
  - libdrm's `drmNmxIoctl` falls back to it on `EPERM`, and ANV opens nodes read-only in that case.
  - (xkpi `48f38b6`; libdrm 0002; Mesa 0002.)
- **App Sandbox profiles deny `/dev/dri` entirely.** They allow IOKit user clients only with an `IOAccelerator`
  provider, and a client on NMXGPU is refused even when opened through NMXAccel.
  - NMXAccel now creates the DRM client itself and attaches it to the accelerator.
  - libdrm's `drmNmxOpen` returns a token fd bound to that client when the node is denied.
  - `drmGetDevices2` describes the render node through it.
  - ANV opens nodes with `drmNmxOpen`.
  - Calculator, Notification Center and other sandboxed apps used to log "zero physical devices". They now get the
    GPU.
  - (NMXAccel `8493109`; libdrm 0004; Mesa 0008.)
- **WebKit's GPU process is stricter still.**
  - It denies `pipe()` and reading `/dev/null`, so the token falls back to a read-only open of libdrm's own file.
    (libdrm 0005)
  - It maps executable code only from the system volume. I installed the GPU bundles under
    `/System/Library/CoreServices/CoreTypes.bundle/Contents/Library/Nullmoth/`, a firmlink into the data volume,
    with `/Library/GPUBundles/*` as symlinks. That's an interim layout.

## Part 4: a GPU-composited desktop in a VM

This needed a display that WindowServer will composite to with Metal. The building blocks come from Nullmoth's
NVIDIA path (display machine, display pipes, AGDC). The display is new: **NMXVGAFB**, an IOFramebuffer on
virtio-vga, which NMXAccel adopts as head 0.

- **Getting CoreDisplay to use the GPU path at all.**
  - `UseIOPresentment()` requires each IOFramebuffer's class name to name its vendor, so the class is
    `NMXIntelVGAFB`.
  - NMXVGAFB must sit in the IOFramebuffer match category, and the VF itself is claimed. Otherwise IONDRV adds a
    second, pipe-less display and CoreAnimation asserts.
  - WindowServer builds display pipes from AGDC. **NMXAGDC** is generated from Nullmoth's NVRMAGDC and answers for
    the VF.
- **Arming without races.** WindowServer starts before the AuxKC kexts load. It then waits for a quiet registry
  before opening displays, and registering a head before `open()` panics.
  - NMXVGAFB holds the registry busy until NMXAccel's first phase is done (gates, transactions, AGDC).
  - The second phase runs inside `NMXVGAFB::open()`, so the pipe exists before WindowServer's open returns.
  - The boot-time WindowServer comes up GPU-composited, with no restarts and no sysctls.
- **"VRAM".** The display family wires display surfaces into video memory before presenting. NMXVGAFB answers
  `nvAllocVram` with physically contiguous system memory.
  - Once guest memory is fragmented, a contiguous 20-37 MB allocation fails. Slots for the largest mode are now
    reserved at start, which fixed a black screen after toggling fullscreen.
- **Zero copy.** NMXVGAFB includes a small virtio-gpu 1.x transport.
  - A presented IOSurface's own pages become a virtio-gpu blob (cached per surface), and a present is a
    `SET_SCANOUT_BLOB` to it. There's no CPU copy and no scan-out buffer.
  - Blob scatter lists go up to 16,000 entries; a 2880x1710 surface is 4,810 pages.
- **No software colour pass.** CoreDisplay ran its colour pipeline (`CombineLUTs` on all cores) for every present:
  - First because AGDC capability 0x711 wasn't answered for CSC, gamma and linearization.
  - Then because WindowServer's accelerated-state machine stuck at `To_Accelerated`. It completes only after a
    reconfiguring connect change. NMXVGAFB now re-advertises the current mode once after the first shown frame.
  - WindowServer at the idle login window went from 105-130% to 40-50% of a core.
- **Frame ordering.** At first, WindowServer waited for the GPU twice per frame.
  - Now flips happen on a thread call once the plugin has copied the frame. The plugin sets IOSurface values
    `NMXPending` at commit and `NMXPublished` after copy-out, and NMXAccel flips when Published ≥ Pending.
  - The IOSurface lock seed can't be used: it moves when a write lock is *taken*.
- **Pacing.** QEMU patch 0007 answers a *fenced* resource flush of a blob scan-out when GTK shows the frame, at the
  latch point of its refresh.
  - NMXVGAFB's flips return there, and its VBL is a time-constraint kernel thread on that grid.
  - The VBL waits in steps, because kernel timer coalescing made it about 1 ms late.
  - QEMU patch 0008 caches the dmabuf per blob resource. GTK had re-imported a 20 MB scattered udmabuf on every
    flip, which cut a flip from 2.4 ms to 37 µs.
- **Mode and cursor.**
  - The mode is the host window's size, with timing from QEMU's EDID (patch 0002 fits 120 Hz modes in the base
    block).
  - Resizing the window sends a display event, and macOS switches to the new size.
  - The pointer is a hardware cursor on virtio-gpu's cursor queue, so pointer motion costs no flips.
- **Control queue.** A single batch is in flight at a time. A display-info query used to overwrite an in-flight
  flip, which froze the display after a fullscreen toggle.

## Part 5: performance and stability

- **"Below 1 fps after a while."** TTM tries order 10..1 allocations with `__GFP_NORETRY` before single pages.
  xnu's contiguous allocator walks physical memory under the page-queue lock, and once memory is fragmented each
  failed try took about 350 ms. High-order `NORETRY` allocations are now refused, and an 8 MB BO create went from
  742 ms to under 1 ms. (`b9acec2`)
- **Login hang.**
  - The port's `sched_pin` only disables interrupts, so an RCU reader that blocked could migrate. `rcu_read_unlock`
    then ended the wrong per-CPU epoch record.
  - drm-kmod also called `synchronize_rcu()` for every new ioctl thread.
  - RCU read sections now end on the record they began on, and drm-kmod skips the call for FreeBSD's integer pids.
  - (xkpi `4d28dc5`, `7f411d0`; drm-kmod 0001.)
- **Panics after WindowServer restarts and process exits.**
  - The port's per-thread and per-process records are keyed by `thread_t`/pid, which xnu reuses. A new thread
    inherited a dead process's mm. (`622c9e7`, `1dc855e`)
  - mm records didn't hold a task reference. (`01ea823`)
- **OpenGL.** Apple's renderer falls back to CPU vertex processing whenever the plugin can't build a pipeline:
  - Minecraft's terrain shader loads an `ivec3` from a float-typed UBO slot.
  - The entity shaders select samplers through a switch on a stored constant.
  - Two translator fixes took Minecraft from 2 to 32 fps. (`56132fc`, `1376c3a`)
  - The GL renderer's primitive-restart and provoking-vertex SPI removed a compute pass per draw.
- **App start-up.** Mesa's optional Vulkan entrypoints became 10,908 flat-namespace imports under
  `-undefined dynamic_lookup`. dyld searched every loaded image for each, so ANV took 0.5-3.7 s to load per process.
  On Apple they now resolve to local weak stubs, and `dlopen` takes 5-10 ms. (Mesa 0007)
- **Smaller wins.**
  - Pooled batch BOs map through one map of their slab parent. (Mesa 0005, 0006)
  - IOSurface textures are images over the surface pages, with no copy in or out.
  - Retired descriptor sets are reused.
  - The plugin's binding mask skips `[[color(n)]]` bindings. Before, most WindowServer pipelines wrote all ~290
    bindings per set.

## Patches

Each directory under `patches/` is a `git format-patch` series. `patches/SERIES.md` lists the base commit of each and
how to apply it.

| Series | Patches | Applies to |
|---|---|---|
| `nvidia-macos-driver` | 47 | [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver) `d96eaca` (2.0.2): NMXVGAFB, NMXAGDC, NMXAccel, plugin, translator |
| `xkpi` | 50 | the same tree's `gpu-port/xkpi` (NMXGPU: xe, LinuxKPI-on-xnu, build in the guest) |
| `drm-kmod` | 1 | [freebsd/drm-kmod](https://github.com/freebsd/drm-kmod) `c16de0f9` |
| `libdrm` | 5 | libdrm `b97cbde1` + Nullmoth's `libdrm-macos.patch` |
| `mesa` | 8 | Mesa `bffd91947ef` (26.3-devel) + Nullmoth's `mesa-macos.patch` |
| `qemu` | 4 | QEMU 11.1.2 |

A few xkpi commits are marked `TEMP` or `diag`. They add diagnostics I haven't removed yet.

### Opting in

- Boot args: `nmxintel=1 nmxvgafb=1`.
- `-nvoff -nmxvgafboff` turns both off.
- NMXGPU's attach boot guard still parks the GPU on the next boot if an attach hangs or panics.
- `nmxvgafb_hz` sets the refresh rate (default 120).
- NMXGPU needs `/Library/NullMoth/firmware/xe/ptl_guc_70.bin` and `ptl_huc.bin`.

### Building

- NMXGPU, the kexts, the plugin and the translator were built inside the macOS VM itself (Command Line Tools,
  Apple clang). See `gpu-port/xkpi/build-guest.sh`.
- NMXGPU can't be injected by OpenCore ("Vtable patching failed"). NMXGPU, NMXAccel, NMXVGAFB and NMXAGDC are
  installed into the Auxiliary KC with `kmutil`, as Nullmoth's own installer does.
- Mesa and libdrm were cross-built on the Linux host with a Meson cross file and the guest's SDK.

## Known issues

- **No VF published to GPUWrangler.** If NMXVGAFB is opted in but NMXGPU isn't driving the VF (e.g. parked by the
  boot guard), the login window waits about 20 s for the display hold. GPUWrangler then waits 60 s for a VF that
  never publishes. Then the CPU desktop comes up.
- **No HiDPI.** Scaled modes need the framebuffer to offer scaling.
- **One CPU copy per frame remains.** The plugin copies its private image into the presented IOSurface.
- **No video decode.** There is no VideoToolbox decoder for Xe3, so HEVC is decoded in software. The animated
  Aerial wallpaper costs more than a core.
- **Leftover work.**
  - The job-timeout (TDR) rewrite from newer kernels isn't backported.
  - One interrupt leak per boot remains (diagnosed and guarded).
  - Some temporary diagnostics remain.
- **Scope.** All of this was tested only on this VM and this host. The changes for the VM display (NMXVGAFB, the
  virtio-gpu flips) are VM-specific. The xe, LinuxKPI and sandbox fixes are not.

## Credits

- Nullmoth's driver tree, including NMXGPU, NMXAccel, the Metal plugin and the translator, is the work of the
  Nullmoth authors. This is a set of changes on top of it.
- Linux `xe` is Intel's; drm-kmod is FreeBSD's; Mesa, libdrm and QEMU are their projects'.
- Each series keeps its upstream's license.
