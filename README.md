# SubSplit DCTL

A subtractive split-toning DCTL for DaVinci Resolve Studio. SubSplit modifies color by subtracting opposing channels rather than increasing values directly, producing subjectively richer, more "filmic" results than typical approaches.

## Features

- **Subtractive Split Toning:** 12 independent sliders for Red, Green, Blue, Cyan, Magenta, and Yellow in both shadows and highlights. Subtractive math mimics film density for subjectively "deeper" colors (or at least doesn't make the image brighter).
- **Customizable Pivot & Transition:** Define the luminance pivot point separating shadows from highlights, and the transition softness between them.
- **Luminance Preservation:** Optional gain compensation that counteracts the darkening inherent to subtractive color adjustments, maintaining perceptual brightness.
- **Show Curve:** Overlays the RGB color manipulation curves directly onto the viewer.
- **Show Pivot:** Displays the pivot point and transition zone on the image.
- **Show Ramp:** Renders a monochrome gradient across the frame showing the effect of the current split-tone settings across the full tonal range. The ramp position is adjustable via a slider, useful for anamorphic or letterboxed footage.

**Note:** By default the pivot is set to the DaVinci Intermediate transfer function's 18% value (0.336 or 33.6%), however the variable pivot allows tuning to your working space's mid-gray if you're using something else.

## Installation

**Note**: DCTLs only work in the Studio (paid) version of DaVinci Resolve.

1. Download `SubSplit.dctl`.
2. Place it in your DaVinci Resolve LUT directory:
   - **macOS:** `Library/Application Support/Blackmagic Design/DaVinci Resolve/LUT`
   - **Windows:** `C:\ProgramData\Blackmagic Design\DaVinci Resolve\Support\LUT`
   - **Linux:** `/home/resolve/LUT`
3. Restart DaVinci Resolve.

> **Note:** Resolve caches compiled DCTL shaders (which are specific to the GPU archecture detected at runtime) independently of the DCTL source itself. When updating a DCTL that invokes a non-visible shader, a full restart of Resolve is required to pick up changes (i.e. the OpenFX panel's "Refresh" option reloads the DCTL source from disk but does not recompile shaders, at least as of Resolve 20.3.2).

## Usage

1. Add a serial node in the Color Page.
2. In the OpenFX panel, search for **DCTL** and drag it onto the node.
3. Select `SubSplit` from the DCTL drop-down in the Inspector.
4. Adjust shadows and highlights using the color sliders.
