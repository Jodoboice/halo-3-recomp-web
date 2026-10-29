# Web port (end goal)

End goal of this fork: the full game running in a browser tab, from the
user's own disc dump (nothing from the retail game is hosted or shipped).
This is a long project. The desktop port comes first, and desktop work
should not make the web port harder.

## Blockers (as of the first assessment)

| Area | Desktop today | Needed for the browser |
|---|---|---|
| Guest memory | one 4 GiB `mmap` reservation; physical memory aliased three times via `memfd` (`memory.cpp`) | single wasm linear memory (memory64 or a compacted guest map); the three physical views need address translation, not page aliasing |
| Fibers | `ucontext` (`hooks/fibers.cpp`) | worker-thread handoff or stack switching; Asyncify over ~22k functions is not realistic |
| Threads | host threads for guest threads, GPU command processor, XMA | Web Workers + SharedArrayBuffer (needs cross-origin isolation headers) |
| GPU | SDL_GPU / Vulkan, GLSL -> SPIR-V with shaderc at runtime | WebGPU backend, WGSL (or SPIR-V -> WGSL) shader path, no shaderc; vertex-stage storage buffers and compute (memexport) need checking against WebGPU limits |
| Audio | SDL3 + FFmpeg XMA decoder | AudioWorklet output, FFmpeg built to wasm |
| Files | host files in `private/game`, `private/hdd` | user's dump read via File System Access API / Blob slices, saves in OPFS; all file calls become async-safe or run in a worker with sync access |
| Code size / speed | native x64 | wasm module for ~6.9 MB of PPC code is very large; compile time and runtime speed are open questions |

## Rules for desktop work meanwhile

- Keep platform calls (memory reservation, fibers, threads, files, window,
  audio) behind small interfaces instead of spreading them further.
- Do not add new dependencies on x86 intrinsics or Vulkan-only features
  outside the files that already have them.
- Prefer changes that move code toward `H3Native/` (host layout, no guest
  memory): every function that leaves the recompiled island is one less
  thing to port to wasm.

## Build reproducibility (blocking everyone, found while taking over)

The `thirdparty/XenonRecomp` submodule is pinned to upstream, but the
recompiler needs local patches (12 PPC instructions, `blrl`, vector pack
formats, jump table tail calls, the `be<T>` change, `Image::dataSize`).
They only exist as `private/xenonrecomp-halo3.patch` on the original
author's machine. Without them `h3xexinfo` does not compile and the
recompiled code would differ from what the project was tested with.
Open item: get the patch from the author, or publish a fork of
XenonRecomp and point `.gitmodules` at it.

## Stages

1. Desktop port solid (rendering bugs, performance, native milestones).
2. Platform interfaces (memory, fibers, threads, files, audio).
3. Emscripten build of the tools and pure libraries (translator, tag
   cache), run in a browser.
4. WebGPU renderer and WGSL shader path.
5. Guest memory and threads in wasm; boot to the main menu.
6. Full game, saves, performance work.
