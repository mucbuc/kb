# Multichannel Signed Distance Fields (MSDF)

## Background: why plain SDFs round off corners

An ordinary signed distance field (SDF) stores, at each texel, the distance from that point to the nearest edge of a shape, with the sign indicating inside vs. outside. This is great for scalable rendering (text, vector icons) — you can zoom, outline, glow, or soften the shape cheaply.

The problem appears near sharp corners. Close to a corner, the true distance function is the minimum (for a convex corner) or maximum (for a concave corner) of two *affine* distance functions, one for each of the two edges meeting at the corner. That min/max produces a crease (a sharp ridge/valley) in the distance field.

If you sample this field at a coarse grid and store a single scalar per texel, that crease information is lost between samples. Bilinear interpolation then blends the two underlying affine planes smoothly, which rounds the corner off — the reconstructed shape looks like it has a small fillet where there should be a sharp point.

## What MSDF does

MSDF stores **three** signed distance values per texel, packed into the R, G, B channels of the texture. Each channel is computed against a different subset of the shape's edges:

1. The shape's contour is split into edge segments at corners.
2. Each edge segment is assigned a "color" — a non-empty subset of {R, G, B} — such that edges on either side of any corner receive *different* color assignments. (msdfgen implements several heuristics for this: "simple," "inktrap," "distance.")
3. For each channel independently, the texel's value is the signed distance to the nearest edge segment carrying that channel's color.

At render time:
- Sample the texture normally (standard hardware bilinear filtering works fine).
- Compute `median(R, G, B)` — this is the effective distance value.
- Apply a smoothstep/threshold based on the screen-space derivative of that median for antialiasing.

Because each individual channel remains a smooth affine field on either side of a corner, taking the median after interpolation reconstructs the crease correctly — corners stay sharp.

## Benefits

- **Sharp corners at any scale.** Crisp corners survive heavy magnification from a single, relatively small texture.
- **Smaller textures / atlases.** Much lower resolution than a plain SDF needs for equivalent corner fidelity, saving memory and generation time.
- **Cheap effects for free.** Outlines, glows, soft shadows, weight (bold/thin) adjustments work the same as with plain SDFs, just without the corner artifacts.
- **Hardware-friendly.** Standard bilinear texture filtering is sufficient; the shader only adds a `median()` and a `smoothstep()`.
- **Mature tooling.** msdfgen (reference implementation) and msdf-atlas-gen (batch font atlas generation) make this practical to integrate into existing pipelines.

## How to create one

1. **Get the vector outline** — line segments and Bezier/quadratic/cubic curves describing the contour (e.g., a font glyph).
2. **Edge coloring** — split the contour at corners and assign each edge segment a color from {R, G, B} (or a pair of channels), ensuring adjacent edges around a corner differ in their assignment.
3. **Per-texel distance computation** — for each output texel and each channel, compute the signed distance to the nearest edge segment carrying that channel's color.
4. **Pack and scale** — write the three distances into R, G, B, scaled to fit the texture's value range (commonly via a "pixel range" parameter that controls how many pixels of distance map to the full channel range).
5. **Render-time reconstruction** — sample with normal bilinear filtering, take `median(R, G, B)`, then smoothstep/threshold using the local screen-space derivative for antialiasing.

Standard tools: **msdfgen** (Viktor Chlumský's CLI/library) and **msdf-atlas-gen** for generating glyph atlases.

## Is MSDF needed if you also have the vector to the nearest point?

This depends on what "vector to nearest point" means in your pipeline.

### Case 1: Per-texel gradient (direction to nearest boundary point) stored alongside the distance

For a true SDF, the gradient ∇d is a unit vector pointing away from the nearest boundary point, and a straight edge's SDF is exactly affine: `d(x) = d0 + ∇d · (x − x0)`. With both distance and gradient stored per sample, you *could* reconstruct the local affine plane each sample belongs to, and near a corner extrapolate two samples' planes to the query point and take their min/max — conceptually similar to what MSDF's median achieves via edge coloring.

So it's not strictly required in a mathematical sense, but in practice it doesn't save much:

- **Storage is the same.** Distance + 2D gradient = 3 values per texel, same as MSDF's three channels.
- **You lose simplicity.** Standard bilinear filtering can't do the plane-extrapolation/min-max step for you — you'd need custom shader logic with raw access to the filter footprint's texels.
- **The gradient field is itself discontinuous across a corner's bisector.** Naively interpolating raw gradient vectors (without an extrapolate-then-combine step) produces nonsensical intermediate directions. You still need to know which edges/features each sample belongs to and whether to take min or max (convex vs. concave corner) — information MSDF's edge coloring encodes implicitly.

### Case 2: Exact analytic access to the vector geometry (not a discretized per-texel value)

If you can query the true distance and nearest point directly against the original curves at any position — no discretized texture involved — then yes, MSDF becomes unnecessary. The interpolation-rounding problem MSDF exists to solve simply doesn't arise, because there's no discretized field being interpolated. This is the approach used by purely geometric GPU text/vector renderers that evaluate curves directly rather than sampling an SDF texture.

### Bottom line

The gradient/vector-to-nearest-point is genuinely useful extra information — and might be worth storing anyway if you need it for other effects (e.g., edge-aware lighting/normals) — but as discretized per-texel data it doesn't eliminate the corner-rounding problem "for free." You'd be trading MSDF's simple median-based reconstruction for a custom plane-extrapolation scheme with comparable storage cost and more shader complexity. If, on the other hand, you have true analytic access to the vector geometry, MSDF is indeed redundant.
