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

## RetroArch

### LCD Shaders

Based on my testing, the ```simpletex_lcd.slang``` shader performs excellently with non-integer scaling (to maximize the gameplay area). While the default settings aren't strictly realistic, they provide a very pleasing aesthetic. Since there are no built-in options to simulate the specific colorimetry of each handheld, this must be handled at the core level. Although dedicated shaders exist for this purpose, they would require creating a separate preset for every console.

In the provided presets, motion blur is handled by ```response-time.slang``` to take full advantage of 120Hz displays. Consequently, any similar options in the core should be disabled. To ensure perfect pixel geometry without introducing shimmering during scrolling, I have placed ```sharp-shimmerless.slang``` at the beginning of the shader chain.

I have created two primary presets:
- [lcd-nolit.glsl](shaders/lcd-nolit.slangp) designed for consoles like the original Game Boy, which feature non-backlit reflective screens
- [lcd-backlit.glsl](shaders/lcd-backlit.slangp) designed for consoles like the GBA (SP/Micro), which utilize backlit displays

For the backlit static parameters, use the following settings as a baseline:

| **Parameter**            | **GBA**               | **DS**              | **PSP**                 |
| ------------------------ | --------------------- | ------------------- | ----------------------- |
| **Grid Intensity**       | **0.40**              | **0.25**            | **0.15**                |
| **Grid Width**           | **0.50**              | **0.40**            | **0.20**                |
| **Darken Colours**       | **0.20**              | **0.15**            | **0.10**                |
| **Darken Grid**          | **1.00**              | **1.00**            | **1.00**                |
| **Background Intensity** | **0.00**              | **0.00**            | **0.00**                |

When it comes to remanence, I would recommend :

| **System**        | **LCD Response Time** |
| ------------------ | ---------------------- |
| **Game Boy (GB)**  | **0.67**               |
| **Game Boy Color** | **0.44**               |
| **GBA (Moyenne)**  | **0.33**               |
| **Nintendo DS**    | **0.22**               |
| **PSP**            | **0.33**               |

### CRT Shaders

#### Global approach

For CRT-based consoles, the Guest shader suite provides the optimal baseline. Given a 1080p handheld display, the "Fast" version is sufficient and prioritizes battery life. The core reference is crt-guest-advanced-fast.slangp.

Two preset families were developed:
- Static-only: Focused strictly on mask structure and scanlines.
- Motion-enhanced: Identical logic but incorporating a crt-beam-simulator pass to improve motion clarity on 120Hz displays.

The following engineering constraints were maintained:
- Non-integer scaling at 1080p: Artifact-free performance across various source resolutions.
- Minimalist processing: Strictly limited to a shadow mask (for dithering treatment) and scanlines where applicable.
- Color accuracy: Colorimetry remains consistent with the raw, unshaded output.

The scanline strategy involves widening the gaps between lines—including in bright areas—to mitigate scaling artifacts. To preserve overall luminance, the scanline intensity is slightly increased (lighter blacks).

### Core resolution enhancements

Furthermore, I dislike the "clinical" look of legacy 3D games rendered at ultra-high resolutions; I find the imbalance between low-fidelity textures/polygons and high-precision rendering to be jarring. Consequently, my policy has been to create:

- **Presets designed for ```1x``` core output**, allowing the Guest shader to handle "organic" interpolation via masks and scanlines (intended for Arcade, SNES, etc.).
- **Presets for low-resolution 3D consoles (PS1, N64)** rendered at ```2x``` by the core and then processed by Guest to add scanlines matching the native vertical resolution. This minimizes wobble and reduces aliasing while preserving consistency between 3D and 2D elements.
- **```Noscan``` shaders** for high-resolution content that apply the mask only, omitting scanlines (Dreamcast games, for instance, should typically not be rendered with scanlines).
- **Flycast-specific shaders** to address the core's lack of anti-aliasing. These ```SSAA``` presets assume a 2x core output (typically 960p), but Guest downscales to 1x before re-sampling with masks for a more analog feel. This is ultimately a 2x SSAA

### Sega consoles

Additionally, Guest includes a parameter specifically for Sega consoles (up to the Saturn) to correct brightness levels. While I haven't created a dedicated preset for these systems, the ```Sega Brightness Fix``` parameter should be set to ```1.0``` for all of them.

### Parameters

> Better trust the ```slang``` files because I'm unsure I kept track of every single change. However, these are the settings corresponding to my approach, so you get the idea.

All the shaders are available here : [shaders](shaders)

#### CRT 1x with scanlines

> Base shader
>
> Files : [crt-shadow1x.slangp](shaders/crt-shadow1x.slangp) 

| Category          | Parameter                          | Value      | Comments                                   |
| ----------------- | ---------------------------------- | ---------- | ------------------------------------------------ |
| **COLOR TWEAKS**  | Contrast Adjustment                | **0.00**   | Maximum color fidelity.                          |
| **COLOR TWEAKS**  | Saturation Adjustment              | **1.15**   | Compensates color loss from mask.                |
| **COLOR TWEAKS**  | Sega Brightness Fix                | **1.00**   | Only for Sega 8/16-bit consoles.                 |
| **GAMMA OPTIONS** | Gamma Input                        | **2.40**   | Standard CRT reference.                          |
| **GAMMA OPTIONS** | Gamma out                          | **2.20**   | Ideal balance for AMOLED.                        |
| **INTERLACING**   | Interlace Trigger Resolution       | **400.00** | Threshold for automatic high-res switching.      |
| **INTERLACING**   | Interlace Mode: OFF...             | **0.00**   | OFF: full progressive image.                     |
| **FILTERING**     | Horizontal sharpness               | **4.00**   | Organic dithering blend.                         |
| **FILTERING**     | Substractive sharpness             | **1.20**   | Locks bidirectional bleed.                       |
| **FILTERING**     | Scanline Spike Removal             | **1.00**   | Vertical stability (non-integer).                |
| **BRIGHTNESS**    | Bright Boost Dark Pixels           | **1.50**   | Recovers shadows under mask.                     |
| **SCANLINE**      | Scanline Shape Dark Pixels         | **2.30**   | Wide and grey scanline gaps.                     |
| **SCANLINE**      | Scanline Shape Bright Pixels       | **1.40**   | Thickens the black core in high-luminance areas. |
| **SCANLINE**      | Scanline Falloff                   | **0.60**   | Defined edges without moire.                     |
| **SCANLINE**      | Increased Bright Scanline Beam     | **0.85**   | Maintains structure in whites.                   |
| **CRT MASK**      | CRT Mask: 0:CGWG, 1-4:Lottes, 5... | **6.00**   | Lottes/Shadow Mask base.                         |
| **CRT MASK**      | Mask Strength (0, 5-12)            | **0.60**   | Balanced texture vs brightness.                  |
| **CRT MASK**      | CRT Mask Boost                     | **1.40**   | Boosts simulated phosphors.                      |
| **CRT MASK**      | CRT Mask Size                      | **1.00**   | Maximum 1080p density.                           |
| **CRT MASK**      | (Transform to) Shadow Mask         | **1.00**   | Circular organic pattern.                        |
| **CRT MASK**      | Mask Layout: RGB or BGR...         | **1.00**   | Physical AMOLED alignment.                       |
| **CRT MASK**      | Lottes maskDark                    | **0.90**   | Brighter mask for global luminance.              |
| **CRT MASK**      | Lottes maskLight                   | **1.60**   | Glow of active phosphors.                        |
| **CRT MASK**      | Smooth Masks in bright scanlines   | **1.00**   | Eliminates banding on white.                     |
| **DECONVERGENCE** | Post Brightness                    | **1.60**   | Final gain for AMOLED pop.                       |

#### CRT without scanlines 

> Modifications on top of CRT 1x with scanlines
>
> File name : [crt-noscan.slangp](shaders/crt-noscan.slangp) and [crt-beam-noscan.slangp](shaders/crt-beam-noscan.slangp) and [crt-ssaa-beam-noscan.slangp](shaders/crt-ssaa-beam-noscan.slangp) and [crt-ssaa-noscan.slangp](shaders/crt-ssaa-noscan.slangp)

| Category          | Parameter                  | Value    | Comments                       |
| ----------------- | -------------------------- | -------- | ------------------------------------ |
| **SCANLINE**      | No-scanline mode           | **1.00** | Disables scanning, keeps mask.       |
| **BRIGHTNESS**    | Bright Boost Dark Pixels   | **1.00** | Neutral: preserves AMOLED blacks.    |
| **BRIGHTNESS**    | Bright Boost Bright Pixels | **1.00** | Avoids clipping on full image.       |
| **COLOR TWEAKS**  | Contrast Adjustment        | **0.00** | Maximum fidelity for 3D textures.    |
| **DECONVERGENCE** | Post Brightness            | **1.25** | Parity with 1x version brightness.   |
| **CRT MASK**      | CRT Mask Boost             | **1.20** | Subtle texture without excess noise. |
| **INTERLACING**   | Interlace Mode: OFF...     | **0.00** | OFF: full progressive image.         |

#### CRT Beam Simulator

> Modifications on top of any spatial shader to deal with temporal aspects
>
> File names : [crt-beam-shadow1x.slangp](shaders/crt-beam-shadow1x.slangp) and [crt-beam-noscan.slangp](shaders/crt-beam-noscan.slangp) and [crt-ssaa-beam-noscan.slangp](shaders/crt-ssaa-beam-noscan.slangp)

| Category           | Parameter                    | Value    | Comments                                  |
| ------------------ | ---------------------------- | -------- | ------------------------------------------------- |
| **BEAM SIMULATOR** | Brightness vs Clarity        | **0.75** | Tradeoff between light and motion blur reduction. |
| **BEAM SIMULATOR** | Gamma                        | **2.40** | Matches the content's encoded gamma curve.        |
| **BEAM SIMULATOR** | LCD Anti-Retention On/Off    | **0.00** | **OFF**: Disables rolling band .                  |
| **BEAM SIMULATOR** | Raster Position Mod          | **0.00** | Sets the beam scan start to neutral position.     |
| **BEAM SIMULATOR** | Scan Direction (0 = No Scan) | **1.00** | Standard horizontal raster scanning.              |

#### CRT 2x with scanlines

> Modifications on top of CRT 1x with scanlines
>
> File names : [crt-beam-shadow2x.slangp](shaders/crt-beam-shadow2x.slangp) 

| Category        | Parameter                     | Value    | Comments                                 |
| --------------- | ----------------------------- | -------- | ------------------------------------------------ |
| **INTERLACING** | Internal Resolution Y: 0.5... | **0.50** | Divides line density to simulate 240p look.      |
| **INTERLACING** | High Resolution Scanlines     | **1.00** | **MANDATORY**: Activates vertical res filtering. |

#### CRT 2x to 1x with antialiasing

The shaders are [crt-ssaa-beam-noscan.slangp](shaders/crt-ssaa-beam-noscan.slangp) and [crt-ssaa-noscan.slangp](shaders/crt-ssaa-noscan.slangp)
