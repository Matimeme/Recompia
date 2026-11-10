# Shared GPU memory: opt-in ReBAR shadow experiment

The goal is to reduce Windows per-process **shared GPU memory**, which may be charged for guest pools imported through `VK_EXT_external_memory_host`, not simply reduce the `[nvn] pool_mirror_mb` VRAM cap.

## What this branch changes

The existing `[nvn] pool_copy = on` path avoids importing ordinary guest memory pools into Vulkan. It instead keeps CPU-visible Vulkan shadows and copies guest bytes into them when needed; GPU writes are landed into guest memory on submission completion. This path was designed for Proton and has known in-flight write/overlap limitations.

An opt-in flag `[nvn] pool_copy_rebar = true` asks the allocator for shadows with **all three** memory properties: `DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT`, preferably the ReBAR-mapped local VRAM type reported by NVIDIA. If a mapped VRAM allocation fails, the path logs the failure and falls back to ordinary host-visible system shadows. If both fail, existing host-import fallback may be used.

This is **not** Native's sparse buffer allocator and **does not** free the game's original guest RAM. The guest still needs its CPU-visible memory; the goal is to stop exposing gigabytes of guest RAM as *Vulkan imported host memory*.

## Test on Windows

Build branch `experiment/shared-memory-rebar-shadows` with the existing manual Release workflow, and extract to a **separate directory**. Retain a working game install and saves backup.

In CMD, navigate to the extracted directory containing `PokopiaLauncher.exe` (replace the path with your real directory), then run:

```bat
cd /d C:\PokopiaRecomp-0.0.0
set NXRT_NVN_POOL_COPY=on
set NXRT_NVN_POOL_COPY_REBAR=1
set NXRT_NVN_POOL_COPY_CELL_MB=16
set NXRT_NVN_POOL_MIRROR_MB=
set NXRT_NVN_POOL_MIRROR_CHUNK_MB=
PokopiaLauncher.exe
```

The existing `nxrt.ini` may override defaults, so remove any conflicting `[nvn] pool_copy`, `pool_copy_rebar`, or `pool_copy_cell_mb` keys for a clear test. Environment variables take priority over the ini according to nxrt configuration.

**Find these lines in `pkp.log`:**
- `nvn.pool: pool_copy on (on)`
- `nvn.pool: pool_copy: shadow placement experimental ReBAR VRAM`
- `vk: shadow buffers (pool_copy, ReBAR VRAM): memory type ...` with `device-local`

If `mapped ReBAR VRAM is unavailable or exhausted` appears, the allocator has fallen back to host memory. Inspect `pool_copy: ... import_fallback` statistics too.

**Compare the same scene, camera path and graphics settings** against the prior stable 2 MiB mirror build (with `pool_copy` left on auto). Record Windows Task Manager **pkp.exe dedicated GPU memory**, **shared GPU memory**, average FPS, 1%-low FPS, worst hitch and graphical correctness. Allow textures/shaders to warm before collecting data.

## Roll back immediately if needed

This opt-in path can be **slower** than the imported-host path: NVIDIA CPU-visible VRAM is often write-combined/uncached, so the CPU's frequent copies or reads of GPU-written bytes may cost significant PCIe bandwidth and CPU time. Pool-copy's own CPU/GPU race limitations are independent of this change.

For the previous behavior in a fresh CMD launch:

```bat
set NXRT_NVN_POOL_COPY=off
set NXRT_NVN_POOL_COPY_REBAR=0
```

Or launch the earlier stable installation. Closing the game and reopening from the previous version does not modify saves; nevertheless back them up before experimenting. The setting is read at startup.

**Status:** source change committed. No Windows compilation or game benchmark has been performed from this environment.
