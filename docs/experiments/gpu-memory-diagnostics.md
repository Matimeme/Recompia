# Experimental GPU memory diagnostics (Windows / Vulkan)

This branch starts with **observation**, not an allocation policy change. The baseline game remains unchanged unless the opt-in diagnostic is enabled.

## Why

The guest's NVN memory pool is system-memory-backed and imported using `VK_EXT_external_memory_host`. Faster GPU Memory (`[nvn] pool_mirror = on`) mirrors select GPU-read buffers into device-local memory. A ReBAR-capable GPU **does not automatically move those original host allocations into VRAM**.

Windows Task Manager reports per-process dedicated and shared GPU memory, but those figures alone do not prove poor allocation or identify what is being accessed over PCIe.

## First test

1. Leave **Faster GPU Memory enabled** in F1 settings.
2. In the active runtime's `nxrt.ini`, add:

   ```ini
   [nvn]
   pool_import_memory_diag = true
   ```

   Or set `NXRT_NVN_POOL_IMPORT_MEMORY_DIAG=1` in the launch environment.
3. Restart the game, load the same scene and watch `pkp.log`.
4. Search the log for `host import memory type`. For each imported host-memory type the code logs:
   - Selected Vulkan memory-type flags and heap.
   - Bitmask of memory types allowed by `vkGetMemoryHostPointerPropertiesEXT`.
   - Whether *any* `DEVICE_LOCAL | HOST_VISIBLE` type exists and whether one is eligible for importing this exact host allocation.
5. Capture FPS, frame-time spikes and the dedicated/shared GPU-memory columns for `pkp.exe`. Compare against a clean upstream build under the same conditions.

## Interpretation

- `any_device_local_host_visible=yes` means a possible CPU-visible local VRAM memory type exists (often associated with ReBAR). It does **not** mean host memory can be remapped into it.
- `compatible_device_local_host_visible=no` means the *imported host pointer* cannot use one of those local-visible types, even if the GPU has them.
- `compatible_device_local_host_visible=yes` is not proof that an imported RAM allocation physically lives in VRAM.
- Forcing a different Vulkan memory type outside the host-pointer-compatible bitmask would be invalid.
- A genuine residency change would require a different allocation/copy/synchronization design, validated for both CPU and GPU writes.

## Scope and safety

This adds one opt-in log message per *selected* imported Vulkan memory type. It does not change the default mirror cap, virtual memory mapping, imported memory type selection, draw path, synchronization or FPS limit.

**Status:** Source change committed. No Windows build or performance benchmark has yet been run on the RTX 3070 Laptop GPU.

## Next investigation

Check log results and frame times before considering small, guarded modifications to mirror residency/copy policy. Always retain an unmodified build for A/B testing.


## Experimental on-demand mirrors (branch `experiment/native-style-on-demand-mirrors`)

This branch adapts **two limited ideas** from Pokopia Native's Vulkan memory system:
- Small, independently backed device-local buffer chunks (2, 4, 8, 16, 32 or 64 MiB).
- For vertex/index/uniform draw bindings, postpone the first chunk allocation until a later submission actually copies requested pages, instead of allocating on the first request even when no copy follows.

It **does not** implement Native's `VK_BUFFER_CREATE_SPARSE_BINDING_BIT` / `vkQueueBindSparse` path. Recompia still imports the original CPU-backed pool via `VK_EXT_external_memory_host`, and a draw whose binding crosses a chunk boundary still reads from that host pool. Smaller chunks can increase fallback reads across PCIe, Vulkan object counts, and driver overhead.

### Comparison commands (Windows)

Launch from the directory containing `PokopiaLauncher.exe` and `pkp.exe` in **each** test, preserving exactly the same scene, texture quality, render resolution, FPS limiter and driver configuration. Do not overwrite a working installation.

**Baseline: 64 MiB chunks (original granularity):**

```bat
cd /d C:\PokopiaRecomp-0.0.0
set NXRT_NVN_POOL_MIRROR_MB=
set NXRT_NVN_POOL_MIRROR_CHUNK_MB=64
PokopiaLauncher.exe
```

**Experimental: 2 MiB chunks (Native-inspired granularity):**

```bat
cd /d C:\PokopiaRecomp-0.0.0
set NXRT_NVN_POOL_MIRROR_MB=
set NXRT_NVN_POOL_MIRROR_CHUNK_MB=2
PokopiaLauncher.exe
```

`NXRT_NVN_POOL_MIRROR_MB=` removes a previous environment override for that CMD session; remove `pool_mirror_mb` from `nxrt.ini` as well if you set it there. Leave Faster GPU Memory **on**.

Confirm the selected size by finding `nvn.pool: pool mirror chunk size: 2 MiB` in `pkp.log`; find `pool mirrors:` for allocation totals. Measure **per-process** dedicated and shared GPU memory for `pkp.exe` in Windows Task Manager, average FPS, 1% low, and longest hitch, ideally after the same load and in the same location.

**Caveat:** This branch is a first-stage experiment in a long-term switch to Native-style sparse residency; it does not yet reduce host-memory imports or promise lower total GPU memory in Windows. Keep the existing 64 MiB path if small chunks substantially hurt FPS.
