
# Math-Art Interfaces & Examples

## Tools discovered (Conradi's stack + alternatives)

- **Python + NumPy + Matplotlib** — Simone Conradi's (profconradi.com) baseline stack for polynomial/complex-plane math art. No generative AI — hand-coded.
- **Marimo** — reactive Python notebook; cells re-run based on data dependencies, not position. Pure-Python `.py` files, git-friendly, deployable as apps/scripts. Requires Python 3.10+. Base install is minimal; `marimo[recommended]` adds duckdb, altair, polars, sqlglot, openai, etc. Does NOT bundle numpy/pandas/matplotlib — installed separately.
- **Apple MLX** — used by Conradi for heavier simulation work (e.g. Particle Lenia).
- **molab** (molab.marimo.io) — free cloud-hosted marimo playground, no install, can sync notebooks from GitHub, runs in-browser via WebAssembly/Pyodide.

## Alternatives more art/math-targeted than engineering-targeted

- **p5.js Web Editor** (editor.p5js.org) — JS-based, built for artists/beginners, huge community precedent for orbit/complex-plane/polynomial art.
- **Shadertoy** (shadertoy.com) — GLSL fragment shaders, live in-browser, large gallery of math-driven visuals, forkable.
- **Hydra** (hydra.ojack.xyz) — live-codable visual synth, chainable function API (`osc().color().modulate().out()`), analog-video-synth mental model, compiles to GLSL under the hood.
- **Dwitter** — 140-byte JS animations; raw JS + Canvas 2D, no shared vocabulary with other tools, extreme-constraint code-golf/math-art hybrid.
- **Desmos** — pure math exploration with sliders, supports recurrence-style definitions, zero code.

## Portability / "universal API" landscape

- **ISF (Interactive Shader Format)** — closest real standard: annotated GLSL fragment shaders + JSON metadata for inputs (sliders, audio-reactive params). Works across VDMX, Resolume, TouchDesigner, Magic Music Visuals. Shadertoy shaders are a short hop to ISF; Hydra sketches can be extracted as compiled GLSL and converted to ISF via editor.isf.video, but Hydra doesn't write ISF natively.
- **Hydra's own API** — chainable, synth-like, easy to improvise in, but Hydra-specific/non-portable.
- **Dwitter** — no standard API; "universal" only in the trivial sense of being plain JS.
- No single standard unifies Hydra-style chaining + Dwitter-style raw JS + GLSL — ISF is the closest shared compilation target.

## Implementation-independent math notation ("lingua franca")

The vocabulary layer used to *describe* this genre of art regardless of renderer:

1. **Recurrence relations** — `x_{n+1} = f(x_n)`. Vocabulary: orbit/trajectory, fixed point, attractor, basin of attraction, bifurcation. Classic example: logistic map `x_{n+1} = r·x_n(1−x_n)`.
2. **Complex dynamics** — `z_{n+1} = z_n² + c` over the complex plane (Mandelbrot/Julia sets). Escape time → color.
3. **Iterated Function Systems (IFS)** — table of affine transforms `{w_1...w_n}` each with probability `p_i`, applied randomly to a point repeatedly. Fully specified by the transform table alone (e.g. Barnsley fern).
4. **L-systems** — axiom + production rules (e.g. `F → F+F−F−F+F`), string-rewritten N times, interpreted via turtle graphics.
5. **Cellular automata** — Wolfram rule numbers (e.g. "Rule 30") fully determine 1D CA behavior from an 8-entry lookup table.

Common thread: each is an implementation-independent formalism specifying *behavior*, leaving rendering as a separate decision.

## Universal pseudo-code abstraction

```
System {
  state:        initial condition(s)
  params:       fixed parameters/config
  step(state, params) -> new_state
  stop(state, history) -> bool
  observe(state, history) -> value      // what gets mapped to visuals
}

run(System, n_max):
  history = [System.state]
  while not System.stop(state, history) and len(history) < n_max:
    state = System.step(state, System.params)
    history.append(state)
  return history
```

Each formalism above = a different flavor of `state`/`step()`/`observe()`. Renderer (Python, Hydra, Shadertoy, ISF) is the only thing that changes once something's expressed this way.

## Composing systems together

- **Sequential piping** — output of system A configures system B's params (e.g. L-system-drawn skeleton → anchor points for an IFS).
- **Nested/hierarchical** — one system's `step()` internally runs another to completion (e.g. a recurrence relation where each value is itself a Mandelbrot escape-time result).
- **Parallel/merge** — run two systems independently, blend their `observe()` outputs (e.g. cellular automaton noise field × logistic-map brightness curve — same pattern as Hydra's `.modulate()`).
- **Feedback loop** — a system's output feeds back as next-frame params (e.g. a Julia set's `c` driven frame-to-frame by a separate orbit — the standard "morphing Julia set" animation technique).

All four are really just "choosing where the wire goes" between `state`, `params`, and `step()` — conceptually identical to Hydra's chaining or ISF's multi-pass shaders, one abstraction level up.
