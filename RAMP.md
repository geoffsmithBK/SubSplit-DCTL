# "Show Ramp" Feature Debugging Log

**Goal**: Display a horizontal gradient ramp across the bottom 15% of the frame when the "Show Ramp" checkbox is activated in the DCTL UI, allowing users to visualize the exact mathematical effect of the split-tone curves across the tonal range.

**Status**: Resolved. The ramp renders correctly with a user-adjustable vertical position.

---

## Attempts & Logic Changes

### 1. Direct Assignment & Overlays (`rgb = ramp`)
* **What I did**: Created a `DCTLUI_CHECK_BOX` called `opt_showramp`. At the bottom of the `transform` function, I added an `if (opt_showramp == 1)` block that simply reassigned the output vector `rgb` to the internal `ramp` vector (`rgb = ramp`).
* **Why it failed**: I originally placed this block *above* the "Show Curve" visualization code. The "Show Curve" logic evaluated the pixel coordinates and, because it processes the whole image, overwrote the pixels I had just assigned to `ramp`.
* **Fix**: Moved the `if (opt_showramp)` block to the very end of the function, right before `return rgb;`.
* **Result**: Still failed to render.

### 2. Evaluating Checkbox State (`if (opt_showramp)`)
* **What I did**: Suspected DaVinci Resolve's DCTL parser might be returning a non-1 truthy value when the checkbox is checked. Changed the conditional statement from `if (opt_showramp == 1)` to just `if (opt_showramp)`.
* **Result**: Still failed to render.

### 3. Stripping Emojis & Viewport Limitation
* **What I did**: Removed all unicode emojis from the `DEFINE_UI_PARAMS` labels across the entire script. DCTL parsers can be notoriously fragile regarding special characters, sometimes causing UI elements to display but fail to bind to their underlying C variables. I also added the logic to constrain the ramp to the bottom 15% of the viewport (`if (y_norm <= 0.15f)`).
* **Result**: Still failed to render.

### 4. Explicit Channel Assignment (`rgb.x = ramp.x`)
* **What I did**: Researched known-working DCTL scripts on GitHub (specifically `LCS-VSP Split Tone V1.2.dctl`). Discovered that when applying a full vector replacement at the end of a script, developers explicitly assign the individual color channels (`x`, `y`, `z`) rather than equating the `float3` vectors (`rgb = ramp`).
* **Implementation**:
```c
if (opt_showramp == 1) {
    float y_norm = 1.0f - (float)p_Y / (float)p_Height;
    if (y_norm <= 0.15f) {
        rgb.x = ramp.x;
        rgb.y = ramp.y;
        rgb.z = ramp.z;
    }
}
```
* **Result**: Still failed to render.

---

## Root Cause: Resolve's Silent Shader Caching

None of the code changes above were the problem. The ramp logic was correct from attempt #4 onward. The actual root cause was **DaVinci Resolve's shader caching behavior**:

1. Resolve parses `DEFINE_UI_PARAMS` declarations as **text metadata**, separately from shader compilation. This means UI elements (sliders, checkboxes) always appear in the inspector, even when the compiled shader is stale or failed to compile.
2. When a modified DCTL file is copied into the LUT directory, Resolve **does not always recompile the shader**. It may continue executing a previously cached compiled version.
3. This creates a deeply misleading situation: the UI reflects the new file (new checkboxes appear, labels update), but the running shader is the old version. New parameters like `opt_showramp` exist in the UI but have no corresponding variable in the executing shader, so toggling them does nothing.

**The fix**: After copying a modified DCTL into the LUT directory, **restart DaVinci Resolve** to force a clean shader recompilation. The standard "Refresh" option in the LUT panel is not sufficient to clear the compiled shader cache.

Once this was understood, the original ramp code (attempt #4) worked immediately without any modifications.

## Final Implementation

The ramp now includes a "Ramp Position" slider (0.0–0.85) allowing the user to move the ramp band vertically, which is useful for anamorphic or letterboxed footage where the default bottom position may be obscured by black bars.
