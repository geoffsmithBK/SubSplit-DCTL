# TODO

## Preserve Luminance: Variable Strength

Convert the current `Preserve Luminance` checkbox into a slider (0.0–1.0, default 1.0) so users can dial in partial luminance compensation rather than a binary on/off. A value of 0.0 would apply no compensation (current unchecked behavior), and 1.0 would apply the full progressive gain curve (current checked behavior). The existing `calculate_luminance_compensation` return value would be interpolated toward 1.0 by the slider amount: `compensation = 1.0 + (compensation - 1.0) * strength`.
