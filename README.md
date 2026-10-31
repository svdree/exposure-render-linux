# Exposure Render 1.1.0 — Linux port

Exposure Render is an interactive CUDA volume renderer implementing *Direct
Volume Rendering with Physically Based Lighting* (T. Kroes, TU Delft). It
progressively path-traces a medical volume (`.mhd`/`.raw`) on the GPU and
displays the converging image in a Qt/VTK desktop application.

![Screenshot](Images/screenshot.png)

The original 2011 release targeted Windows with Visual Studio 2010, CUDA 4.0,
Qt 4.7 and VTK 5.6. None of those can be installed today, so the code has been
ported to a current Linux toolchain:

| | Before | Now (verified against) |
|---|---|---|
| Compiler | MSVC 2010 | GCC (15.3) |
| Build | CMake 2.8 + `FindCUDA` | CMake 3.20+ (4.4.2), CUDA as a language |
| CUDA | 4.0, `compute_20` | 12.0+ (13.3), `native` architecture |
| Qt | 4.7 | 6 (6.11.2) |
| VTK | 5.6 | 9 (9.6.2) |

Rendering has been verified end to end on an RTX 2060 with all three bundled
examples, covering every shading mode (BRDF, phase function and hybrid).

See [build.md](build.md) to build and run it.

## What changed

The port was done in small steps, one commit each, so the history reads as a
changelog. In order:

1. **Line endings normalised to LF.** Whitespace only; `.gitattributes` keeps
   it that way and marks volume data and icons binary.
2. **MSVC-only constructs removed.** Extra member qualifications, `sprintf_s`
   and friends, `__declspec` exports, MSVC `#pragma`s, backslash includes and a
   non-UTF-8 byte that GCC refused to parse. No behaviour changes.
3. **Build system rewritten.** The CMake file now uses CUDA as a first-class
   language, component-based `find_package` for Qt 6 and VTK 9, AUTOMOC in place
   of hand-maintained moc lists, and `target_precompile_headers` scoped to C++
   so Qt never reaches nvcc. About half the size of the original.
4. **Qt 4 → Qt 6.** Mechanical replacements for removed APIs (`toAscii`,
   `qSort`, `setResizeMode`, Qt3Support leftovers, …). `QHttp` is gone, so the
   HTTP helper was reimplemented on `QNetworkAccessManager`.
5. **VTK 5.6 → VTK 9.** `QVTKWidget` → `QVTKOpenGLNativeWidget` with an explicit
   `vtkGenericOpenGLRenderWindow`; `SetInput` → `SetInputData`.
6. **CUDA 4.0 → CUDA 12/13.** Texture references (removed in CUDA 12) became
   texture objects held in constant memory, so kernels address them by the same
   names. Other removed calls replaced: string-symbol `cudaMemcpyToSymbol`,
   `cudaMemcpyToArray`, `cudaThreadSynchronize`, `cudaDeviceProp::clockRate`,
   and the old SDK's `cutil_math.h`.

Once it ran, a handful of run-time problems surfaced and were fixed:

- **Two startup crashes.** An out-of-bounds read in the presets widget that
  Qt 4 happened to tolerate and Qt 6 does not (a bug predating the port), and a
  missing default surface format that left VTK without a usable OpenGL context.
- **Volume paths were mangled.** Every path had `/` rewritten to `\\`, so no
  volume could load on Linux.
- **Command-line loading.** `ExposureRender <volume.mhd>` now opens a volume
  directly, skipping the startup dialog; `--help` and `--version` work too.
- **Per-frame GPU allocations removed.** Two fixed-size device objects were
  allocated and freed on every frame; they are now allocated once.
- **Dead update check disabled.** It called a Google Code URL that has not
  existed since 2016. It now runs only if `updates/versionInfoUrl` is set in
  QSettings, and the *Check for updates* menu item is back.
- **Frame buffer race fixed.** An unnamed `QMutexLocker` temporary released its
  lock immediately, and the render thread never locked at all, so the UI could
  copy a buffer while it was being reallocated. Both sides now lock properly.

The Windows code paths are preserved but have not been built since the port.

## Further reading

- [build.md](build.md) — dependencies, build steps and troubleshooting.
- [CLAUDE.md](CLAUDE.md) — architecture notes and known rough edges.
- `git log` — each port commit explains what it changed and how it was verified.
