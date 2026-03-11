# TODO

## 1. Dual-Axis Blend (Color Strength / Density Strength)

Replace or supplement the Global Blend with two independent sliders: `Color Strength`
and `Density Strength`. Color Strength scales the chroma component of all shadow/
highlight adjustments; Density Strength scales the luminance compensation independently.
This exposes the two fundamentally different things this DCTL does (hue shift vs.
subtractive density) as separate creative axes, which is not possible with Resolve's
single Global Blend lerp. A luminance-weighted variant of either axis (where the effect
fades faster in highlights than in shadows, or vice versa) would be a natural extension.

## 2. Three-Way Grading (Add Midtone Zone)

Add a midtone adjustment zone with its own set of RGB + CMY sliders, positioned at the
pivot point. The midtone weight would be highest at the pivot and fall off symmetrically
into the shadow and highlight regions using the existing smooth_step logic. This extends
the tool from two-way to full three-way shadow/midtone/highlight grading within the
subtractive framework, without any additive math.

## 3. Transition Curve Shape

Add a `Transition Shape` slider that morphs the shadow/highlight boundary from the
current smooth-step (S-curve) toward a linear ramp or a harder, more abrupt threshold.
Colorists matching to film scans or specific log curves often need a non-S-shaped
transition — a log gamma like ARRI LogC or DaVinci Intermediate doesn't distribute
tonal values evenly, and the shape of the transition matters for where the effect
"lives" in the image. Implement as a blend between smooth_step and a pure linear
clamp: `t = lerp(linear_t, smoothstep_t, shape)`.

## 4. Per-Channel Highlight Soft Clip

Add optional per-channel soft clipping in the highlight region, modeled after the way
color negative film rolls off gracefully toward saturation. Rather than hard-clamping
at 1.0, a soft clip would apply a toe curve above a user-defined threshold per channel.
This pairs naturally with the subtractive approach: as one channel is emphasized and
the complementary channels are suppressed, the dominant channel can clip harshly without
it. Implement as a per-channel hyperbolic rolloff above a `clip_threshold` param.

## 5. Saturation Compensation

Mirror the existing Luminance Preservation logic but for saturation. The subtractive
adjustments can desaturate the image in regions where opposing channels are suppressed
simultaneously (e.g., pulling both cyan and red in shadows cancels out colorimetrically).
Calculate the predicted saturation change from the combined adjustments and apply an
optional compensating saturation boost, with a 0.0–1.0 strength slider analogous to
Preserve Luminance. Would need to operate in a polar chroma space (e.g., derive
saturation as distance from the achromatic axis in RGB).

## 7. Preserve Skin Tones

A hue-angle holdout mask that protects skin tones from being affected by the split-tone
adjustments. In DaVinci Resolve's vectorscope (BT.709, Cr/Cb axes), the skin tone
indicator sits at approximately 123–127° in the Cb/Cr plane — the "11 o'clock" position
— which corresponds to a hue angle of roughly 20–28° in standard HSV (0° = red,
60° = yellow). Blackmagic does not publish the exact value, but 24° is a reasonable
center anchor for a first implementation.

Conceptually the mask is a "fuzzy pie slice" emanating from the neutral axis: a 2D
region in hue-saturation space that is widest at moderate saturations (where skin lives)
and tapers toward both the neutral axis (where hue is undefined) and high saturations
(where the hue read becomes more reliable and less skin-like). Implementation sketch:

1. Derive HSV inline from RGB: `v = max(R,G,B)`, `s = (v - min(R,G,B)) / v`,
   `h = atan2-based hue in [0, 360)`.
2. Compute signed angular distance from skin center: `d = abs(hue - 24.0)`, wrapping
   at 180° to handle the red/magenta boundary correctly.
3. Compute a hue-based mask with user-controllable bandwidth:
   `hue_mask = 1.0 - smoothstep(0, bandwidth, d)` where bandwidth grows with the slider
   (e.g. `bandwidth = 15.0 + preserve_skin * 20.0` degrees).
4. Attenuate near the neutral axis (undefined hue) using saturation:
   `sat_gate = smoothstep(0.05, 0.20, s)`.
5. Final holdout: `skin_mask = hue_mask * sat_gate`.
6. Blend: `output = lerp(adjusted, original, skin_mask * preserve_skin)`.

At slider 0.0 the mask is off and the full effect applies everywhere. At 1.0 skin-toned
regions are fully held out from the adjustment. The angular bandwidth grows with slider
value, creating the "fuzzier pie slice" at higher protection levels. A future refinement
could expose bandwidth as a separate slider or add a separate shadow/highlight weighting
so skin protection is stronger in one tonal range than another.

## 8. Preserve Neutrals

A chroma-based holdout mask that attenuates the split-tone effect in pixels whose RGB
channels are close to achromatic — protecting gray ramps, specular highlights, white
walls, and other content that should remain unbiased by the toning operation.

"Neutralness" is most naturally measured by the spread between the maximum and minimum
RGB channel values: `divergence = max(R,G,B) - min(R,G,B)`. This is the numerator of
HSV saturation, but measured in absolute (rather than relative) terms, which behaves
more intuitively in log-encoded footage where absolute channel values carry density
information. A pixel at (0.5, 0.5, 0.5) has divergence = 0.0 (dead neutral);
a pixel at (0.5, 0.43, 0.35) has divergence = 0.15 (at the boundary the user described).

Implementation sketch:

1. `divergence = max(R,G,B) - min(R,G,B)` — computed from the *input* (pre-adjustment)
   pixel to avoid feedback.
2. `neutral_mask = 1.0 - smoothstep(0.0, threshold, divergence)` where `threshold` is
   the user's 15% boundary. Could be hardcoded at 0.15 or exposed as a second slider
   for expert control.
3. `output = lerp(adjusted, original, neutral_mask * preserve_neutrals)`.

One subtlety: in HDR / scene-linear or log footage, the 0.15 divergence threshold may
need to be expressed relative to the pixel's luminance rather than as an absolute value
to avoid treating very dark or very bright neutrals differently. A normalized variant:
`divergence = (max - min) / max(max, epsilon)` (i.e. HSV saturation) avoids this,
at the cost of making very dark near-neutrals look over-protected. The right default
likely depends on the working color space and is worth exposing to the user.

Both `preserve_skin` and `preserve_neutrals` could eventually be combined into a single
"Protect Content" section with a shared visualization overlay (drawn on top of the image
using the existing overlay infrastructure) showing which pixels are being held out and by
how much.

## 6. Chromatic Aberration / Halation (Creative)

A subtle, physically-inspired halation effect: in regions where highlights are bright,
bleed a small amount of the highlight color adjustment into the surrounding pixels with
a falloff based on local luminance gradient. This mimics the way film base fluoresces
red/orange around bright light sources. Implement as a two-pass approach: first pass
computes a per-pixel "bloom weight" from luminance; second pass blends a colored offset.
Note: true multi-pass is not possible in a single DCTL transform — this would require
either a convolution approximation or restructuring as a separate DCTL in a node chain.
