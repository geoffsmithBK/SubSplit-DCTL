# SubSplit DCTL

A subtractive split toning DCTL for DaVinci Resolve. SubSplit modifies color by subtracting opposing channels rather than adding light, producing richer, more filmic results than standard lift/gamma/gain wheels.

## Features

- **Subtractive Split Toning:** 12 independent sliders for Red, Green, Blue, Cyan, Magenta, and Yellow in both shadows and highlights. Subtractive math mimics film density for deeper colors without washing out the image.
- **Customizable Pivot & Transition:** Define the luminance pivot point separating shadows from highlights, and the transition softness between them.
- **Luminance Preservation:** Optional gain compensation that counteracts the darkening inherent to subtractive color adjustments, maintaining perceptual brightness.
- **Show Curve:** Overlays the RGB color manipulation curves directly onto the viewer.
- **Show Pivot:** Displays the pivot point and transition zone on the image.
- **Show Ramp:** Renders a horizontal gradient across the frame showing the exact effect of the current split-tone settings across the full tonal range. The ramp position is adjustable via a slider, useful for anamorphic or letterboxed footage.
- **DaVinci Wide Gamut Tuned:** Luminance coefficients are matched to DaVinci Wide Gamut for accurate results.

## Installation

1. Download `SubSplit.dctl`.
2. Place it in your DaVinci Resolve LUT directory:
   - **macOS:** `Library/Application Support/Blackmagic Design/DaVinci Resolve/LUT`
   - **Windows:** `C:\ProgramData\Blackmagic Design\DaVinci Resolve\Support\LUT`
   - **Linux:** `/home/resolve/LUT`
3. Restart DaVinci Resolve.

> **Note:** Resolve caches compiled DCTL shaders independently of the source file. When updating a DCTL, a full restart of Resolve is required to pick up changes — the LUT panel "Refresh" option does not recompile the shader.

## Usage

1. Add a serial node in the Color Page.
2. In the OpenFX panel, search for **DCTL** and drag it onto the node.
3. Select `SubSplit` from the DCTL drop-down in the Inspector.
4. Adjust shadows and highlights using the color sliders.
