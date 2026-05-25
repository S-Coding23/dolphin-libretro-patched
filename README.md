# Patched Dolphin Libretro Core — Build Instructions

This builds a `dolphin_libretro.dll` with a one-line fix that forces GPU
synchronization after each frame, so libretro frontends that read back the
framebuffer via `glReadPixels` get complete frames instead of black/partial.

## Option A: GitHub Actions (recommended — no local toolchain needed)

1. **Create a new GitHub repo** (e.g. `dolphin-libretro-patched`)

2. **Copy these files into it** preserving the directory structure:
   ```
   .github/workflows/build_win64.yml
   patches/0001-force-glFinish-after-frame.patch
   ```

3. **Push to GitHub:**
   ```
   git init
   git add .
   git commit -m "Add dolphin build workflow with glFinish patch"
   git remote add origin https://github.com/YOUR_USERNAME/dolphin-libretro-patched.git
   git push -u origin main
   ```

4. **Wait for the build** — go to Actions tab, watch it run (~10-15 min)

5. **Download the artifact** — click the completed run, download
   `dolphin_libretro-win64-patched.zip`

6. **Replace your core:**
   ```
   copy dolphin_libretro.dll c:\Rec0m88\cores\win\dolphin_libretro.dll
   ```

## Option B: Local VS2022 build (fallback)

If the Actions build fails or you want to iterate locally:

```cmd
REM Clone
git clone https://github.com/libretro/dolphin.git dolphin-src
cd dolphin-src
git submodule update --init --recursive

REM Apply the patch manually — edit Source\Core\DolphinLibretro\Main.cpp:
REM   1. Add  #include <GL/gl.h>  near the top includes
REM   2. After the block:
REM        if (system.IsDualCoreMode()) { ... } else { ... }
REM      and BEFORE RETRO_PERFORMANCE_STOP, add:
REM        if (g_gfx && Config::Get(Config::MAIN_GFX_BACKEND) == "OGL")
REM        {
REM          glFinish();
REM        }

REM Build
mkdir build && cd build
cmake -DLIBRETRO=ON -DENABLE_QT=0 -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target dolphin_libretro --config Release

REM The DLL will be in build\Binaries\Release\ or similar
```

## What the patch does

In `Source/Core/DolphinLibretro/Main.cpp`, inside `retro_run()`, after
the GPU frame processing (`RunGpuLoop()` / `RunSingleFrame()`), we add
`glFinish()` when the OGL backend is active. This ensures the XFB-to-system-FBO
blit has completed before retro_run() returns, so the frontend's readback
captures the real frame instead of a partially-rendered one.

## If `#include <GL/gl.h>` doesn't compile

The Dolphin codebase may use its own GL header path. Try these alternatives:
- `#include "Common/GL/GLExtensions/GLExtensions.h"`
- Or just call `glFinish()` without the extra include — the OGL headers are
  likely already pulled in transitively via `OGLGfx.h`

Check the build error and adjust accordingly.
