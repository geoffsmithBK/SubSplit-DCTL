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

## 6. Chromatic Aberration / Halation (Creative)

A subtle, physically-inspired halation effect: in regions where highlights are bright,
bleed a small amount of the highlight color adjustment into the surrounding pixels with
a falloff based on local luminance gradient. This mimics the way film base fluoresces
red/orange around bright light sources. Implement as a two-pass approach: first pass
computes a per-pixel "bloom weight" from luminance; second pass blends a colored offset.
Note: true multi-pass is not possible in a single DCTL transform — this would require
either a convolution approximation or restructuring as a separate DCTL in a node chain.
