# RG35XX-H / R36S Porting Notes

## Target

- KNULLI on Anbernic RG35XX-H (Cortex-A53, AArch64 Linux userspace)
- SDL2 software renderer; no OpenGL, OpenGL ES, DRM, or KMS dependency
- SDL2 audio and SDL game-controller input
- PortMaster packaging, persistent save/configuration, and Start + Select exit
- Keep the existing Windows, desktop Linux, and Android targets working

## Current source findings

The desktop Linux build is currently configured in `Makefile_pc`:

- `NATIVE_LINUX=1` selects the native SDL2 path.
- It hardcodes `-m32` in both compiler and preprocessor flags.
- It links SDL2, SDL2_image, OpenGL, and GLU.
- The SDL2 backend only enables SDL game-controller input for Android; Linux currently returns keyboard input only.
- The Linux renderer is created with `SDL_RENDERER_PRESENTVSYNC`, leaving renderer selection to SDL.

The README explains that the current portable engine uses 32-bit pointer assumptions. Therefore, changing `-m32` to AArch64 flags alone is not a valid port. The handheld build must first settle the ABI strategy.

## ABI decision / gate

The target hardware's Cortex-A53 supports AArch32 and AArch64, but the KNULLI userspace is AArch64. Two approaches need validation:

1. **AArch64 native build (preferred if feasible):** audit and remove 32-bit pointer assumptions in the portable engine, then build against the device's AArch64 SDL2/runtime libraries.
2. **AArch32 compatibility build:** cross-compile an ARM hard-float executable with 32-bit pointers and verify that the target KNULLI image supplies a compatible ARMHF runtime and SDL2 libraries.

Do not merge an ARMHF binary or claim it runs on KNULLI until the runtime ABI and SDL2 dependencies have been verified on the device.

## Implementation sequence

1. Resolve the runtime ABI and pointer-width constraint.
2. Add an isolated handheld make target; do not change the existing Windows/Linux/Android build defaults.
3. Make the handheld SDL renderer explicitly use `SDL_RENDERER_SOFTWARE`.
4. Enable SDL game-controller support on the handheld target and map the RG35XX-H controls.
5. Add Start + Select clean exit, runtime logging, and reliable save flushing.
6. Add a PortMaster launcher/package and a manual GitHub Actions workflow.
7. Build and test on KNULLI RG35XX-H; then check R36S/DarkOSRE compatibility.

## Current status

Initial source audit and dedicated `rg35xx-h-port` branch created. No handheld binary has been built or tested yet. The ABI/runtime validation is the first implementation gate.
