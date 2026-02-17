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

This discussion focuses exclusively on home consoles. We are targeting shaders for the Dreamcast and earlier generations. While scanlines don’t really make sense for the Dreamcast, they are essential for older systems. However, shadow masks are relevant for every console because they 'upscale' pixels correctly, introducing more detail and color depth.

We’ve selected crt-guest-advanced-fast for its flexibility and excellent handling of non-integer scaling (as I prefer a full-height display). The 'fast' version is more than adequate; it helps conserve battery life while reducing heat and fan noise. I also pair it with crt-beam-simulator to leverage the 120Hz screen without darkening the display as much as standard Black Frame Insertion (BFI) would.

Early 3D consoles like the N64 or PS1 are often thought to look better with internal resolution upscaling. In my opinion, multipliers beyond 2x look terribly imbalanced, but letting the core upscale to 480p is acceptable. However, to achieve this while maintaining the intended scanline effect, the shader preset requires a few specific tweaks.

> The following is Gemini's

### Comprehensive CRT Calibration for 1080p OLED Handhelds

This finalized configuration is designed for a handheld console (1080p OLED, 5.5", 120Hz). It combines the temporal motion clarity of **crt-beam-simulator** (Pass 0) with the spatial signal granularity of **crt-guest-advanced-fast** (Main Pass). The settings ensure absolute robustness against **non-integer scaling**, atypical **Arcade PAR**, and aggressive **overscan crops**.

### Temporal Integration: CRT Beam Simulator (Pass 0)

This shader must be **prepended** (placed at the top of the shader list) to simulate the electron gun's temporal behavior.
*Requirement: Set "Shader Sub-frames" to **2** in RetroArch Settings > Video > Synchronization.*

#### CRT-BEAM-SIMULATOR (TEMPORAL BFI)

| Parameter (UI Label) | Recommended Value | Technical Purpose |
| --- | --- | --- |
| `Brightness vs Clarity (GAIN_VS_BLUR)` | **0.65** | Best balance for 120Hz; reduces motion blur while preserving OLED luminosity. |
| `Gamma` | **2.40** | Matches the OLED black point to prevent gray banding or posterization. |
| `LCD Anti-Retention On/Off` | **0.00** | **OFF** for OLED. Prevents micro-stutter and removes unnecessary input lag. |
| `Scan Direction` | **0.00** | Global BFI mode. Preferred for small 5.5" screens to reduce rolling scan fatigue. |

### Mode 1x: Native Resolution Configuration (NES, SNES, Genesis, DC)

*Note: Dreamcast (480p) is handled via the Interlace Trigger to ensure a solid high-resolution image without flickering.*

#### COMMON PARAMETERS

| Section / Parameter (UI Label) | Recommended Value | Technical Reason |
| --- | --- | --- |
| **** |  |  |
| `Sega Brightness Fix` | **1.00 / 0.00** | **1.00 for Genesis/Saturn**; **0.00 for Nintendo/DC** (prevents white clipping). |
| `Saturation Adjustment` | **1.20** | Counteracts desaturation from high Bright Boost levels. |
| `Brightness Adjustment` | **1.05** | Final master gain for linear luminance fine-tuning. |
| **** |  |  |
| `Gamma Input` / `Gamma out` | **2.40 / 2.40** | Standard CRT gamma curve optimized for OLED contrast. |
| **** |  |  |
| `Interlace Trigger Resolution/VGA...` | **375.00** | Detects Dreamcast 480p to disable scanlines for a solid VGA look. |
| `Interlace Mode` | **0.00** | Disables flicker for a stable upscaled high-res image. |
| `Interlace Scanline Effect ('Laced...` | **1.00** | Fills 480p/VGA line gaps to create a solid, non-pixelated texture. |
| **** |  |  |
| `Horizontal sharpness` | **4.60** | Blends dithering patterns while preserving text legibility. |
| `Substractive sharpness (1.0 recom...` | **1.00** | Removes white ringing/halos around sharp pixel edges. |
| `Scanline Spike Removal` | **1.00** | **Critical:** Ensures scanline uniformity during non-integer scaling. |
| **** |  |  |
| `Max. Glow/M.Glow Value` | **0.15** | Subtle analog light leak typical of CRT phosphors. |
| `Horizontal Bloom/Halation Sigma` | **0.80** | Controls the soft spread of light for a "glassy" texture. |
| **** |  |  |
| `Integer Scaling: Odd:Y, Even:'X'+Y` | **0.00** | Allows full screen height and robust overscan cropping. |
| `CurvatureX / CurvatureY` | **0.00** | Keeps the grid flat to prevent moiré on 1080p matrices. |
| **** |  |  |
| `Lottes maskLight` | **1.85** | Weights mask transparency to regain OLED peak brightness. |
| `Bright Boost Dark Pixels` | **1.55** | Heavily boosts dark details to combat BFI/Mask darkening. |
| `Bright Boost Bright Pixels` | **1.15** | Increases highlight punch without washing out scanline structure. |
| **** |  |  |
| `Scanline Shape Dark Pixels` | **1.30** | Prevents scanlines from looking too harsh or thick. |

#### MASK SPECIFIC PARAMETERS

| Parameter (UI Label) | Mask 6 (Pro ~360 TVL) | Mask 10 (Salon ~270 TVL) |
| --- | --- | --- |
| **** |  |  |
| `CRT Mask:` | **6.00** (RGB) | **10.00** (RGBX) |
| `Mask Strength` | **0.75** | **0.90** |
| `CRT Mask Size` | **1.00** (Strict) | **1.00** (Strict) |
| `(Transform to) Shadow Mask` | **1.00** (Active) | **1.00** (Active) |
| `Mask Layout: RGB or BGR...` | **1.00** (OLED-BGR) | **1.00** (OLED-BGR) |
| `Slot Mask Width (0:Auto)` | **0.00** (Auto) | **0.00** (Auto) |
| `Slot Mask Height: 2x1 or 4x1...` | **2.00** (Anti-Moiré) | **2.00** (Anti-Moiré) |

---

### Mode 2x: Core Upscaled Configuration (PS1, N64)

*Forces 240p-style thick scanlines and signal granularity on a 480p internal core buffer.*

#### COMMON PARAMETERS

| Section / Parameter (UI Label) | Recommended Value | Technical Reason |
| --- | --- | --- |
| **** |  |  |
| `Sega Brightness Fix` | **0.00** | Generally not required for 3D-era systems. |
| `Saturation Adjustment` | **1.30** | Boosts model vibrancy for upscaled 3D graphics. |
| `Brightness Adjustment` | **1.10** | Linear gain boost for increased upscaled pixel density. |
| **** |  |  |
| `Interlace Trigger Resolution/VGA...` | **600.00** | Prevents the shader from removing scanlines at 480p. |
| `Internal Resolution Y: 0.5...y-dow...` | **0.50** | **The Secret:** Resamples the 480p signal back to 240p scanline size. |
| `High Resolution Scanlines (prepend...` | **1.00** | Enables correct beam dynamics for doubled core buffers. |
| **** |  |  |
| `Horizontal sharpness` | **3.80** | Blends 2x pixels for an analog, film-like CRT feel. |
| `Scanline Spike Removal` | **1.00** | Fixes Moiré artifacts caused by core upscaling. |
| **** |  |  |
| `Horizontal Bloom/Halation Sigma` | **0.95** | Wider sigmas to "glue" 3D polygon edges into the CRT grid. |
| **** |  |  |
| `Lottes maskLight` | **1.95** | Essential for highlight vibrancy on upscaled content. |
| `Bright Boost Dark Pixels` | **1.70** | Strong boost for deep black depth in 3D scenes (e.g., PS1 horror). |
| **** |  |  |
| `Scanline Shape Dark Pixels` | **1.45** | Widens scanline gaps for an authentic 240p aesthetic. |

#### MASK SPECIFIC PARAMETERS

| Parameter (UI Label) | Mask 6 (Pro ~360 TVL) | Mask 10 (Salon ~270 TVL) |
| --- | --- | --- |
| **** |  |  |
| `CRT Mask:` | **6.00** | **10.00** |
| `Mask Strength` | **0.80** | **0.95** |
| `CRT Mask Size` | **1.00** | **1.00** |
| `CRT Mask Zoom (+ mask width)` | **-1.00** (Fine grain) | **-1.00** (Fine grain) |
| `CRT Mask Zoom Sharpen` | **0.65** | **0.75** |
| `(Transform to) Shadow Mask` | **1.00** | **1.00** |
| `Mask Layout: RGB or BGR...` | **1.00** | **1.00** |
| `Slot Mask Width (0:Auto)` | **3.00** (**Locked Fixed**) | **4.00** (**Locked Fixed**) |
| `Slot Mask Height: 2x1 or 4x1...` | **2.00** | **2.00** |

### Technical Performance & Stability Insights

* **Arcade PAR Robustness:** The analytic beam integration in `guest-advanced-fast` automatically adjusts scanline and mask density to the 1080p viewport. This makes it inherently stable for Arcade games with unconventional aspect ratios (CPS1/2, etc.) or non-square pixels.
* **Shadow Mask Transformation:** Activating `(Transform to) Shadow Mask` (1.00) alongside a `Slot Mask Height` of **2.00** creates a staggered grid. This setup is the most resistant to moiré during non-integer scaling, as it breaks the linear vertical alignment of the phosphors.
* **Brightness Compensation:** Always use the `Bright Boost` and `maskLight` parameters before the master `Brightness Adjustment`. These tools expand luminance intelligently based on local contrast rather than applying a flat, destructive linear gain.
* **Locked Mask Width (Mode 2x):** In Core 2x mode, "Auto" width (0.00) is unreliable. Manually fixing it to **3.0** (Mask 6) or **4.0** (Mask 10) locks the triads to the physical OLED subpixels, ensuring the texture never shifts regardless of internal resolution.
