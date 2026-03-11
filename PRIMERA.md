# PRIMERA

**Working name for an open-source, browser-native, ACES-native node-based color grading environment for independent colorists and the open-source community.**

---

## Vision

Color grading at a professional level requires three things: a rigorous color science foundation, an expressive and well-ordered creative toolset, and hardware access that doesn't discriminate by operating system. DaVinci Resolve provides all three — but only on its own terms, on its own platforms, under its own proprietary execution environment. For the independent colorist working on Linux, for the Kdenlive editor who wants a real grading companion, and for the open-source community that has built robust tools around every other stage of post-production, the grading stage remains a walled garden.

Primera is a proposal to change that.

The project is built on three convictions:

1. **Color science should be open.** ACES is an open standard. OpenColorIO is open. Jed Smith's Open Display Transform is open. The math that governs how a pixel moves from camera through grade to display is not proprietary. The execution environment shouldn't be either.

2. **Opinions are a feature, not a limitation.** The best grading interfaces are opinionated about order of operations. A node graph that lets you do anything in any order is a blank canvas; a node graph that guides you toward best practice while preserving creative freedom is a tool. Primera should be the latter.

3. **The browser is the only universal runtime.** A browser-based application runs identically on Linux, macOS, Windows, and ChromeOS. It requires no installation, no dongle, no driver compatibility matrix. WebGPU, which reached broad browser support in 2024–2025, makes GPU-accelerated pixel processing in a browser tab a viable reality for the first time.

---

## Color Science Architecture

### Working Spaces

| Stage | Color Space | Notes |
|-------|-------------|-------|
| File I/O and archival | ACES2065-1 (AP0, scene-linear) | The ACES interchange format. Widest gamut, linear transfer function. All internal frame storage. |
| Creative grading | ACEScct (AP1, log-like) | The colorist-facing working space. Log-like transfer function provides perceptually even distribution of exposure stops across the slider range, similar to working in camera log. AP1 primaries are tighter than AP0 and map cleanly to Rec.709/P3/2020. |
| Display output | ODT-dependent | See Output Transform section below. |

### The Pixel Bus

Every pixel entering Primera is immediately converted to ACES2065-1 via an Input Device Transform (IDT). This is the internal representation — all frames live as 32-bit float ACES2065-1 in GPU memory. Before entering any grading node, the pixel is converted to ACEScct. After the final node, it is converted back to ACES2065-1 for the output transform. Grading nodes never see or output ACES2065-1; they are entirely ACEScct in, ACEScct out. This separation makes node implementations simple and prevents accidental working-space confusion.

```
[Camera file] → IDT → ACES2065-1 → [to ACEScct] → [Node 1] → [Node 2] → ... → [to ACES2065-1] → ODT → [Display]
```

### Input Device Transforms

IDTs convert camera-native color (e.g. ARRI LogC4/AWG4, RED IPP2/WG, Sony S-Log3/SGamut3.Cine, DJI D-Log M) to ACES2065-1. At launch, Primera will adopt IDTs from the official ACES Central repository (CTL source, transpiled to WGSL). A community IDT contribution workflow should be a first-class concern from the beginning.

### Output Transform: Jed Smith's Open Display Transform

The ACES Reference Rendering Transform (RRT) + Output Device Transform (ODT) pipeline has well-documented aesthetic problems: a slightly desaturated look, aggressive highlight compression, and behavior in the blues that many colorists find unflattering. Several commercial productions have abandoned it in favor of custom picture formation pipelines.

Jed Smith's [Open Display Transform](https://github.com/jedypod/open-display-transform) (OpenDRT / Chromagnon) is an open-source alternative that prioritizes author control and perceptual fidelity. It is available in DCTL and as Nuke nodes, with a C core that is straightforwardly transpilable to WGSL for Primera's WebGPU pipeline. It will serve as the default and preferred output transform, with the standard ACES RRT+ODT available as an alternative for productions that require ACES-compliance certification.

Display targets at launch: sRGB/Rec.709 SDR, P3-D65 (for DCI/Apple displays), and BT.2020 PQ (HDR10). HLG to follow.

---

## Node Graph: Philosophy and Design

### LiteGraph.js

The node graph UI is built on [LiteGraph.js](https://github.com/jagenjo/litegraph.js) (MIT license), the same library that underlies ComfyUI's graph interface. It renders to Canvas2D, supports zoom/pan/multi-select, handles subgraphs, and has proven itself at scale with hundreds of simultaneous nodes. Critically, it is MIT-licensed and actively maintained (v1.0 shipped March 2024, 7.9k stars).

### Order of Operations: Opinionated by Default

The node graph has a concept of **tiers** — logical positions in the grading pipeline that Primera understands and can guide the user toward. Nodes have a declared tier and Primera will warn (not prevent) when a node is connected out of canonical order. This is the opinionated layer: you can override it, but you have to mean to.

The canonical tier sequence for primary corrections:

```
[INPUT]
  │
  ▼
[TIER 1: Balance]       — White balance, temperature/tint correction
  │                       Operates before any tonal work, mirrors the
  ▼                       on-set decision of setting the camera's WB.
[TIER 2: Exposure]      — Global exposure in scene-linear EV units.
  │                       Applied before contrast so the pivot point
  ▼                       of contrast is anchored correctly.
[TIER 3: Contrast]      — Pivot-based contrast (ASC CDL Power, or
  │                       a tone curve variant). Shapes the tonal
  ▼                       relationship of the image.
[TIER 4: Primary Color] — Per-channel balance (ASC CDL Slope/Offset,
  │                       or shadow/highlight color wheels). The main
  ▼                       creative color decision for the base grade.
[TIER 5: Saturation]    — Global and selective saturation. After
  │                       primary color so saturation is evaluated
  ▼                       on the already-toned image.
[OUTPUT]
```

Each node occupies one tier position. You can have multiple nodes in the same tier (e.g. two successive CDL nodes as a rough pass and a fine pass). The enforced rule is simpler than the tier ordering: **each node does exactly one thing.** A "White Balance + Contrast" combined node is not a valid node type in Primera.

### Node Anatomy

A Primera node is a self-contained unit consisting of:

- **A WGSL compute shader** — the pixel transform, ACEScct in, ACEScct out
- **A parameter manifest** — declares UI controls (sliders, checkboxes, color pickers) with types, ranges, and defaults
- **Tier declaration** — where in the OOO this node belongs
- **A display name and description**

This is structurally identical to DCTL (DEFINE_UI_PARAMS + `__DEVICE__ float3 transform()`), but targeting open, standardized WGSL rather than Blackmagic's proprietary execution environment. A DCTL-to-Primera transpiler is a tractable early project — the languages are structurally similar enough that most DCTLs could be mechanically ported.

---

## MVP Node Set: Primary Corrections

### 1. White Balance
- **Parameters:** Color Temperature (Kelvin, 2000–12000K), Tint (green–magenta offset)
- **Implementation:** Convert illuminant color temperature to XYZ using Planckian locus, derive a chromatic adaptation matrix (CAT16 or Bradford), apply in ACEScct. Tint is a green/magenta offset orthogonal to the blackbody locus.
- **Tier:** 1 (Balance)

### 2. Exposure
- **Parameters:** Exposure (EV, –6 to +6, default 0)
- **Implementation:** Convert ACEScct to scene-linear, apply `* pow(2, ev)`, convert back. This is the only primary correction node that temporarily leaves ACEScct — exposure in EV units is only meaningful in linear. The conversion round-trip is cheap.
- **Tier:** 2 (Exposure)

### 3. ASC CDL
- **Parameters:** Slope RGB (0–2, default 1.0), Offset RGB (–0.5–0.5, default 0), Power RGB (0.1–3.0, default 1.0), Saturation (0–2, default 1.0)
- **Implementation:** The ASC Color Decision List transform is the industry's lingua franca for primary grading interchange. `out = clamp(in * slope + offset, 0, 1) ^ power`. Applied in ACEScct.
- **Notes:** CDL values are exportable as `.cdl` / `.ccc` / `.edl` for interchange with Resolve, Baselight, and any other ASC CDL-aware application. This is a core interoperability feature.
- **Tier:** 3–4 (Contrast and Primary Color, depending on which parameters are in use — a future UI refinement could split this into two linked nodes)

### 4. Lift / Gamma / Gain (Color Wheels)
- **Parameters:** Three color wheel pickers (shadow lift, midtone gamma, highlight gain) + luminance offset for each
- **Implementation:** The classic broadcast grading paradigm. Shadow, midtone, and highlight weights derived from luminance using smooth transitions (similar to SubSplit's pivot/transition model). Each zone gets an additive RGB offset from its color wheel. Operates in ACEScct.
- **Tier:** 4 (Primary Color)

### 5. Contrast (Tone Curve)
- **Parameters:** Pivot (0–1, default 0.18 scene-linear equivalent in ACEScct), Contrast (0–3, default 1.0)
- **Implementation:** Applies `(in - pivot) * contrast + pivot` in ACEScct, which is a pivot-based linear contrast in the log domain (equivalent to an S-curve in linear). A spline-based tone curve editor is a post-MVP refinement.
- **Tier:** 3 (Contrast)

### 6. Saturation
- **Parameters:** Global Saturation (0–3, default 1.0)
- **Implementation:** Convert to luminance-preserving saturation in ACEScct: `out = luma + (in - luma) * saturation` where `luma` uses AP1 luminance coefficients (0.2722, 0.6741, 0.0537).
- **Tier:** 5 (Saturation)

---

## Technical Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Node graph UI | LiteGraph.js (MIT) | Proven, ComfyUI-validated, actively maintained |
| GPU compute | WebGPU (WGSL shaders) | Universal browser support by 2025, proper compute pipeline |
| GPU fallback | WebGL2 (GLSL) | For browsers/drivers without WebGPU |
| Color management | OpenColorIO-web (WASM) or bespoke WGSL | OCIO has a WASM build; alternatively, IDTs/ODTs are transpiled directly to WGSL |
| Output transform | OpenDRT (transpiled C → WGSL) | See color science section |
| File I/O | OpenEXR.js (WASM), plus JPEG/PNG via Canvas API | EXR for professional work; JPEG/PNG for accessibility |
| UI framework | Svelte or vanilla JS | Svelte for reactivity without overhead; no React/Vue dependency |
| Distribution | Progressive Web App (PWA) + optional Electron shell | Browser-first, no install required; Electron for local file system access |

### On WebGPU Performance

WebGPU compute shaders process image data in parallel workgroups across the GPU. For a 1920×1080 frame, at 16×16 workgroups, this is 8,100 dispatch calls each processing 256 pixels in parallel — competitive with native compute. 4K (3840×2160) is approximately 32,400 dispatches. Real-time preview at reduced resolution (1/2 or 1/4) is achievable; full-4K real-time is hardware-dependent but feasible on modern discrete GPUs. This is sufficient for a primary-corrections MVP where the node count is low.

---

## Interoperability

- **ASC CDL export** — Every grade expressible as CDL values can export a `.cdl` file importable by Resolve, Baselight, Scratch, and any other CDL-aware tool. This is a critical adoption bridge: colorists can rough-grade in Primera and round-trip into their finishing environment.
- **ACES CLF (Common LUT Format)** — Export the full node chain as an ACES CLF file for application in any OCIO-integrated tool.
- **LUT export** — Bake the grade to a 33×33×33 3D LUT (`.cube`) for delivery to any color-managed environment, with the understanding that LUTs are scene-specific approximations.
- **DCTL export** — A stretch goal: generate a DCTL from the node chain for direct use in DaVinci Resolve. The structural similarity between Primera nodes and DCTL makes this tractable.

---

## Relationship to Kdenlive

Kdenlive (KDE's open-source video editor) is the most capable Linux NLE without a strong color grading story. MLT, which underlies Kdenlive, supports GLSL-based video filters and OCIO integration. The most realistic integration path is:

1. **Standalone Primera grades CDL/CLF** — exported and applied in Kdenlive via an OCIO LUT node. No direct integration required; the interchange format does the work.
2. **Primera as an external process** — Kdenlive passes a frame range to Primera via a shared socket or temp file, Primera returns a graded output. Similar to how DaVinci is used as an external grading step from Premiere.
3. **Native Kdenlive plugin (long-term)** — A Kdenlive video effect that runs Primera's GLSL shaders natively in MLT's pipeline, with Primera's UI hosted in a WebView. This is the most ambitious path but technically achievable given Kdenlive's Qt/WebEngine infrastructure.

---

## Roadmap

### Phase 1: Proof of Concept
- [ ] Single-frame WebGPU pipeline: load EXR → IDT → ACEScct → one shader pass → ACES2065-1 → OpenDRT → display
- [ ] LiteGraph.js canvas with a hard-coded linear node chain (no dynamic graph yet)
- [ ] Implement 3–4 MVP nodes (Exposure, ASC CDL, Saturation, White Balance)
- [ ] OpenDRT transpiled to WGSL and running in browser

### Phase 2: Node Graph
- [ ] Dynamic LiteGraph.js graph: add/remove/reconnect nodes
- [ ] Node parameter UI (sliders, wheels) rendered in node panels
- [ ] Tier ordering system with soft warnings
- [ ] Real-time preview at 1/2 resolution

### Phase 3: I/O and Interoperability
- [ ] EXR import/export (OpenEXR.js WASM)
- [ ] ASC CDL export
- [ ] ACES CLF export
- [ ] 3D LUT (.cube) export
- [ ] IDT library (ARRI, RED, Sony, DJI at minimum)

### Phase 4: Kdenlive Bridge
- [ ] CDL/CLF round-trip workflow documentation
- [ ] Evaluate Kdenlive WebView plugin path
- [ ] Outreach to Kdenlive maintainers

### Phase 5: Beyond Primaries
- [ ] Secondary corrections (HSL qualifiers, masking)
- [ ] SubSplit-style subtractive split toning (this DCTL, ported to WGSL, would be a natural early secondary node)
- [ ] Preserve Skin / Preserve Neutrals (see TODO.md in SubSplit)
- [ ] Hardware panel support (Contour ShuttlePRO, Loupedeck, MIDI controllers via WebMIDI API)
