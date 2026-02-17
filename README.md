# Install notes for the Retroid Pocket 6 handheld console

> Many things from my [RPM Install](https://github.com/gerpy/rpm-install) still stand. Changes mostly come from the larger, 16/9, 1080p and 120Hz screen.

![Consoles vs TV tech](consoles-tv.png)

## Standalone emulators shaders

Wii is the last console for which it makes sense to emulate using CRT shaders. So for NetherSX2 and Gamecube, we use the build-in shaders, which are often not fancy, for the better. Scanlines typically make zero sense on such consoles displaying 480p and not 224p or 240p. Only TV consoles are discussed here.

### Dolphin

Built-in shaders don't help much. Clownacy has ported GLSL shaders for the standalone Dolphin emulator:  
https://clownacy.wordpress.com/2023/06/30/porting-crt-shaders-from-retroarch-to-dolphin/. The latest versions are found in this repo: https://github.com/dolphin-emu/dolphin/tree/master/Data/Sys/Shaders/.

The shaders are `crt-pi.glsl` and `crt-lottes-fast.glsl`. They are supposed to expose parameters to the Dolphin UI, but Dolphin on Android lacks the parameters tweaking feature as on Windows. Therefore, the shaders need manual edits of default values for optimal rendering.

Scanlines produce a lot of moiré/banding artifacts on the RPM, to the point that the shaders become unusable. There shouldn't be visible scanlines at all for GameCube 480p content. To remove scanlines as well as curvature, the shaders need to be modified in an editor by changing their default values. These are modified **`crt-lottes-fast.glsl`** :

- With **mask #1** (aperture grille) : [crt_lottes_fast_mask1.glsl](https://github.com/gerpy/rpm-install/blob/main/Dolphin%20Shaders/crt_lottes_fast_mask1.glsl)

- With **mask #3** (shadow mask) : [crt_lottes_fast_mask3.glsl](https://github.com/gerpy/rpm-install/blob/main/Dolphin%20Shaders/crt_lottes_fast_mask3.glsl)

Shadow masks do a better job at smoothing while Aperture grilles produce a sharper image. I chose **mask #2** here. The `crt-pi.glsl` mask is just an aperture grille with scanlines, without added value over `crt-lottes-fast.glsl` IMO.

### NetherSx2

The built-in Triangle and Lottes shaders scale well. Others produce banding/moiré artifacts. I choose **`Lottes`**, which resembles Dolphin's mask #2.

## RetroArch shaders

Only TV consoles are discussed here. We want shaders for Dreamcast and below. Scanlines make no sense for Dreamcast but they do for earlier consoles. Maks make sense for every console since they "upscale" pixels in the proper way, introducing more details and colors.

