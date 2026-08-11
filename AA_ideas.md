# Anti-Aliasing: Techniques & Ideas

## Standard techniques

**Spatial (per-frame sampling)**
- Supersampling (SSAA) — render bigger, downsample. Brute force, expensive, highest quality.
- Multisampling (MSAA) — only supersamples geometry edges; useless on procedural/SDF shaders (no polygon edge to detect).
- Coverage sampling (CSAA) — cheap MSAA variant.
- Ordered/rotated grid supersampling — fixed subpixel sample patterns instead of random.
- Analytic/exact coverage AA — geometric coverage computation instead of sampling (e.g. font rendering).

**Shader-side (relevant for SDF/procedural art)**
- `smoothstep()` edge softening — replace hard `step(threshold, d)` with a smooth transition.
- Screen-space derivatives (`fwidth`/`dFdx`/`dFdy`) — auto-scale the smoothstep width to pixel size; the "correct" resolution-independent SDF AA.
- Analytic AA of implicit functions — closed-form coverage integration, no sampling noise.

**Temporal**
- TAA — accumulate jittered subpixel samples across frames, reproject with motion vectors; can ghost/smear on fast motion.
- Temporal supersampling — simpler jittered accumulation.
- Motion blur — blurs along the velocity vector; solves "strobing edges in motion" directly.
- Feedback/decay trails — `pixel = max(new, previous * decay)`; softens temporal edges via exponential falloff. Same principle as CRT phosphor persistence and Hydra's self-referencing feedback (`src(o0).blend(...)`).
- Temporal dithering / Frame Rate Control (FRC) — alternates colors *at* a pixel across frames instead of blending spatially; how cheap LCDs fake extra color depth.

**Post-process**
- FXAA — cheap edge-detect + blur, blurs fine detail indiscriminately.
- SMAA — smarter edge-detection AA.
- MLAA — pattern-matches edge shapes and blends.
- DLSS/FSR/XeSS — ML-based temporal reconstruction, hybrid of supersampling + TAA.

**Font/vector-specific**
- Subpixel rendering (ClearType-style) — exploits LCD subpixel layout.
- Hinting — snaps outlines to pixel grid instead of smoothing (sharp-but-correct vs. smooth-but-approximate).

## Temporal dithering — deep dive

- Mechanism: alternate a pixel between two colors across frames so the *probability* of a color simulates an intermediate shade, rather than blending colors spatially.
- This is literally **FRC (Frame Rate Control)**, used by cheap LCD panels to fake extra bit depth.
- Human flicker sensitivity peaks around **15 Hz**, broadly sensitive from **4–30 Hz**. A naive 50/50 flip at 60fps is a 30Hz cycle — right at the edge of the sensitive range.
- Pattern *period* matters more than raw framerate: a dither pattern repeating every 4 frames at 60Hz → visible 15Hz flicker (worst case); repeating every 16 frames → 3.75Hz (much less visible).
- At 60Hz, only ~3–4 frames are usable for temporal averaging before flicker appears — limits how many perceptual "shades" you can fake, which is bad news for smooth gradients specifically.
- **Motion is the failure case**: dithering assumes a pixel stays at the same screen location across frames. A moving edge breaks that assumption, producing shimmer/crawl instead of clean blur — well-documented on FRC panels showing video.
- Better version: **spatiotemporal blue noise** — blue-noise spatial pattern that also shifts/rotates per frame, spreading error across space *and* time. Used in modern real-time ray tracing denoisers. Produces "clean film grain" instead of "crunchy sparkle" on moving edges.

## Feedback/decay trails — deep dive

- `pixel = max(new_value, previous_pixel * decay)` — bright new content snaps on instantly, old content fades exponentially instead of hard-cutting.
- Directly analogous to CRT phosphor persistence.
- Solves motion aliasing by smoothing the *motion*, not the *edge* — conceptually closer to motion blur, but cheaper (one blend equation, one prior-frame buffer, no extra samples).
- For color: fade each RGB channel by the same decay, or fade in a perceptual space (HSL lightness / linear light) to avoid hue-shifting trails.
- Native to Hydra via self-referencing output buffers (`src(o0).blend(newFrame, 0.1).out(o0)`).

## Esoteric / artistic directions (exaggerating or subverting AA)

- **Dithering as aesthetic** — ordered/Bayer, blue-noise, Floyd-Steinberg error diffusion; deliberate visible dot patterns (see: *Return of the Obra Dinn*).
- **Temporal dithering as texture** — tune the flicker to sit in a deliberately "buzzy" band rather than a clean one.
- **Exaggerated smoothstep width** — many-pixels-wide falloff turns edges into soft glow/bloom.
- **Chromatic aberration on edges** — offset the AA falloff per color channel for lens-glass-style fringing.
- **Anisotropic/directional blur tied to motion vector** — stretch blur only along movement direction; stylized motion energy.
- **Posterized/quantized AA** — smoothstep then quantize to a few gray levels — cel-shaded stepped edges.
- **SDF "glow bands"** — `sin(d * frequency)` on the distance field turns the AA zone into rippling concentric contours (topographic-map look).
- **Reverse AA / intentional large-scale jaggies** — quantize coordinates to a coarse grid before computing the SDF — pixel-art-meets-vector look.
- **Stochastic transparency** — per-pixel random discard instead of alpha blending; historically used for order-independent transparency; same core idea as temporal color dithering.
- **Noise-modulated edge width** — feed Perlin/simplex noise into the smoothstep width for a wobbly, painterly ink-line look.

## Mosaic-tile / pixel-as-object idea (brainstormed, not AA per se)

- Reframes pixels from a **recomputed field** (standard shader/raster model — no memory between frames) to **persistent objects with identity and position** that glide toward target slots and spawn/despawn at edges.
- Core difficulty: the **correspondence problem** — when a row grows from 5 to 7 slots, deciding which old tile maps to which new slot/position isn't a free choice (same issue as classic shape-morphing vertex correspondence).
- Existing software pattern for this: **D3.js enter/update/exit** — buckets elements into entering (animate in), updating (animate position delta), exiting (animate out) when underlying data changes shape.
- Physical precedent: **Daniel Rozin's "Wooden Mirror"** (1999/2014) — 830 motorized wood tiles mirroring a live camera feed via rotation; tiles have real inertia, so fast motion causes visible mechanical "catching up." (https://vimeo.com/101408845)
- Other precedent: split-flap/Solari boards, sliding tile puzzles, sorting-algorithm visualizations.
- Code examples to reference: CodePen "Grid Reflow Animation" (https://codepen.io/iamryanyu/pen/RNaLoGQ — smooth per-tile reflow between layouts) and "Animate CSS Grid" (https://codepen.io/aholachek/pen/VXjOPB — animates insertion of new tiles into a live grid).
- Would translate to the state/step/observe framing (see below) as a per-tile system: `state = position`, `params = {target, speed}`, `step()` = ease/spring toward target, special spawn rule for new tiles entering off-screen.

---
