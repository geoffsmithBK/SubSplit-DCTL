# SubSplit DCTL

SubSplit is a Subtractive Split Toning DCTL (DaVinci Color Transform Language) script for advanced color grading in DaVinci Resolve.

## Features

- **Subtractive Split Toning:** Modifies color by subtracting opposing color channels rather than adding light (additive math). This mimics film density, resulting in richer, deeper colors without washing out the image.
- **Granular Color Controls:** Provides 12 independent UI sliders to adjust Red, Green, Blue, Cyan, Magenta, and Yellow independently in both the shadows and the highlights.
- **Customizable Pivot & Transition:** Users can define the exact luminance pivot point separating shadows from highlights, as well as the transition softness between them.
- **Luminance Preservation:** An optional toggle that calculates the darkening impact of the subtractive adjustments and applies a progressive gain compensation curve to maintain perceptual brightness.
- **Visualizations:** Includes checkboxes to overlay visual representations of the pivot point, transition curves, and the RGB color manipulation directly onto the image viewer.
- **DaVinci Wide Gamut Tuned:** Luminance mapping is mathematically tuned to DaVinci Wide Gamut coefficients for accurate and cinematic results.

## Intended Use

This tool is designed for **colorists and video editors working in DaVinci Resolve** who want to build high-end, cinematic split-tone looks (such as the classic teal and orange). By applying it via a DCTL node in the Color Page, you can dial in deep, filmic color separation in the shadows and highlights without experiencing the unwanted brightening or washed-out effect typical of standard lift/gamma/gain color wheels.

## Installation

1. Download the `SubSplit.dctl` file.
2. Place it in your DaVinci Resolve LUT directory:
   - **macOS:** `Library/Application Support/Blackmagic Design/DaVinci Resolve/LUT`
   - **Windows:** `C:\ProgramData\Blackmagic Design\DaVinci Resolve\Support\LUT`
   - **Linux:** `/home/resolve/LUT`
3. Restart DaVinci Resolve or right-click in the LUT panel and select "Refresh".

## Usage

1. Add a new serial node in the Color Page.
2. In the OpenFX panel, search for **DCTL** and drag it onto the node.
3. In the Inspector, select `SubSplit` from the DCTL drop-down menu.
4. Adjust the shadows and highlights to your liking using the intuitive color sliders.
