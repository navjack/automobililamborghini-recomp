# macOS port notes — Automobili Lamborghini: Recompiled

Boots the ROM and renders through Metal at ~28-30 fps with working audio and controller input.
Installed as `AutomobiliLamborghini.app`.

## Changes made

**`CMakeLists.txt`**
- `add_compile_options("SHELL:-include stdlib.h")`. The `SHELL:` prefix is **required** here:
  this project already passes `-include cstdint`, and without `SHELL:` CMake de-duplicated the
  repeated `-include` token into `-include stdlib.h cstdint`, making `cstdint` a bare input file
  and producing a confusing "cstdint not found" error.
- `set(RT64_STATIC TRUE)`, and NFD's `NFD_PORTAL` scoped to non-Apple.

**`src/main.cpp`**
- `SDL_syswm.h` include extended from `_WIN32` to also cover `__APPLE__` (it was gated to
  Windows, so `SDL_SysWMinfo` was an unknown type on macOS).
- Apple branch in `create_window_stub()` returning `{NSWindow*, CAMetalLayer*}`.
- `macos_prepare_bundle_environment()` — detects an .app bundle via `_NSGetExecutablePath`,
  chdirs to `~/Library/Application Support/AutomobiliLamborghini`, and returns the bundled ROM
  path. Three lookups here resolve against the working directory (the default ROM name, the
  syms table, and the controller pak in `libultra_stubs.c:160`), and a Finder launch starts with
  the working directory at `/`.

**`src/lambo_crash.cpp`** — replaced the Linux-only `read_symlink("/proc/self/exe")` with
`_NSGetExecutablePath`. On macOS the original threw an uncaught `filesystem_error` and killed the
process at startup. Also added a `Contents/Resources` candidate for the symbol table, and switched
the Linux path to the non-throwing `error_code` overload.

**`tools/make_macos_app.sh`** (new) — assets are copied from the **build** tree, not the source
tree: `assets/ui/fonts/` in the source tree holds only a README, and CMake's post-build step is
what copies the Lato faces in. Bundling from the source tree silently ships a fontless UI.

## Build

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release .
cmake --build build -j8
./tools/make_macos_app.sh
```

## Data locations

- ROM: bundled in `Contents/Resources` (also accepted as a positional argv path)
- Symbols: `Contents/Resources/lamborghini.syms.toml`
- Controller pak: `~/Library/Application Support/AutomobiliLamborghini`
- Graphics config: `~/.config/LamborghiniRecomp/graphics.json`

## Known issues

One intermittent SIGBUS observed in rt64/plume's Metal backend across four runs (the other three
ran 75-90s clean through the same point). Shared renderer issue, not specific to this port.

## Shared background (applies to every rt64-based port here)

**hlslpp `labs`** — rt64 forces `HLSLPP_SCALAR` on Apple Silicon and its `scalar.h` calls
`labs()` without including `<stdlib.h>`. Fix: `add_compile_options("SHELL:-include stdlib.h")`.
The `SHELL:` prefix matters — without it CMake de-duplicates the repeated `-include` token and
collapses it with any other `-include`, turning the following header into a bare input file.

**Window handle** — ultramodern's `WindowHandle` is platform-variant. On Apple it is
`{NSWindow*, CAMetalLayer*}`, and plume's Metal backend casts `.view` straight to
`CA::MetalLayer*`. Passing the `SDL_Window*` instead crashes in `objc_msgSend` the moment the
render thread starts. Correct form:

```cpp
flags |= SDL_WINDOW_METAL;
SDL_MetalView metal_view = SDL_Metal_CreateView(window);
return ultramodern::renderer::WindowHandle{ wmInfo.info.cocoa.window,
                                            SDL_Metal_GetLayer(metal_view) };
```

**Bundling rules learned the hard way**
- `codesign` treats *any* non-Mach-O file in `Contents/MacOS` as unsigned nested code and
  refuses to seal the bundle. Data files belong in `Contents/Resources`.
- Strip absolute Homebrew `LC_RPATH` entries. These builds bake `/opt/homebrew/lib` in as the
  **first** rpath; dyld searches rpaths in order, so `@rpath/libSDL2-2.0.0.dylib` resolves to
  Homebrew and the vendored copies are never loaded. `otool -L` still looks correct because it
  only shows the `@rpath` load commands — verify with
  `otool -l <bin> | grep -A2 LC_RPATH` instead.
- sdl2-compat `dlopen`s `libSDL3.dylib` at load time. Dependency walkers (including CMake's
  `fixup_bundle`) never see it, so it must be vendored explicitly or the app aborts in
  `dllinit` with "Failed loading SDL3 library" before `main()` runs.
- Test the **installed** copy from `/Applications`, not the build directory. A writable build
  dir hides any bug involving a read-only or relative config path.

**N64Recomp is per-project** — every project pins its own recompiler under `lib/N64Recomp` or
`lib/N64ModernRuntime/N64Recomp`. Build *that* copy and run it from the repo root (the build
shells out to a literal `./N64Recomp <game>.toml`). Reusing another project's binary appears to
work and emits a plausible file count, but the generated code targets a different runtime API
and fails much later with errors like `no member named 'nullopt' in namespace 'std'` inside
`recomp_overlays.inl`. Regenerate after `rm -rf RecompiledFuncs RecompiledPatches`.

**Known upstream issue (not fixed)** — rt64/plume's Metal backend over-releases metal-cpp
objects. It shows up as a crash on quit in the "RT64 Workload" thread inside `objc_release` /
`objc_autoreleasePoolPop`, with memory-poison values (`0xa1a1a1a1`, `0xa3a3a3a3a3a3a3a3`) in the
registers. Adding a per-iteration `NS::AutoreleasePool` to rt64's render thread loop makes it
**worse** — the crash moves to ~0.5s after launch and macOS reports "API MISUSE: Over-release of
an object", proving the missing pool was masking a genuine double-release rather than causing
it. Do not re-attempt that fix; the real bug is unbalanced retain/release inside plume.
