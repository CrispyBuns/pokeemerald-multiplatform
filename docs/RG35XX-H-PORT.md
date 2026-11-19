# RG35XX-H / R36S Porting Notes

## Target

- 32-bit ARM Linux (AArch32 / ARMHF), targeting the Cortex-A53
- KNULLI on Anbernic RG35XX-H; keep R36S/DarkOSRE compatibility in view
- SDL2 software renderer; no OpenGL, OpenGL ES, DRM, or KMS dependency
- SDL2 audio and SDL game-controller input
- PortMaster packaging, persistent save/configuration, and Start + Select exit
- Keep the existing Windows, desktop Linux, and Android targets working

## ABI decision

The handheld port is **32-bit ARM hard-float (ARMHF)**, not AArch64. The intended compiler target is `arm-linux-gnueabihf`, with Cortex-A53 tuning (for example, `-mcpu=cortex-a53`). The executable must use 32-bit pointers and link only against compatible 32-bit ARMHF libraries.

The Cortex-A53 can execute AArch32, but the KNULLI image must also provide a compatible 32-bit ARMHF userspace/runtime and SDL2 libraries. This is a hard deployment requirement: do not package or describe the binary as runnable until the target's loader, libc, SDL2, and SDL2_image availability have been checked. If the KNULLI image does not provide that runtime, the port needs a self-contained/compatible runtime strategy rather than silently switching to AArch64.

## Current source findings

The desktop Linux build is currently configured in `Makefile_pc`:

- `NATIVE_LINUX=1` selects the native SDL2 path.
- It hardcodes `-m32` in compiler and preprocessor flags, but this is the desktop host's 32-bit x86 ABI, **not ARMHF**.
- Its assembler flags include `--32`, also an x86 assembler mode; these cannot be reused for an ARM cross-build.
- It links SDL2, SDL2_image, OpenGL, and GLU.
- The SDL2 backend only enables SDL game-controller input for Android; Linux currently returns keyboard input only.
- The Linux renderer is created with `SDL_RENDERER_PRESENTVSYNC`, leaving renderer selection to SDL.

The ARMHF handheld build therefore needs its own isolated make target/toolchain configuration. It must use the ARM cross-compiler and matching ARM assembler, and must not inherit the x86 `-m32` / `as --32` setup.

## Implementation sequence

1. Verify that the target KNULLI image can run ARMHF binaries and has (or can be packaged with) compatible 32-bit SDL2 dependencies.
2. Add an isolated ARMHF handheld make target; do not change the existing Windows/Linux/Android build defaults.
3. Use Cortex-A53 tuning and a 32-bit ARM hard-float toolchain; keep the ABI explicit and verify the resulting ELF with `file` and `readelf`.
4. Make the handheld SDL renderer explicitly use `SDL_RENDERER_SOFTWARE`.
5. Enable SDL game-controller support on the handheld target and map the RG35XX-H controls.
6. Add Start + Select clean exit, runtime logging, and reliable save flushing.
7. Add a PortMaster launcher/package and a manual GitHub Actions workflow.
8. Build and test on KNULLI RG35XX-H; then check R36S/DarkOSRE compatibility.

## Current status

Initial source audit and dedicated `rg35xx-h-port` branch created. The ABI decision is 32-bit ARMHF, but no handheld binary has been built or tested yet. Runtime compatibility validation is the first implementation gate.
