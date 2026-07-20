# Development Notes

Handoff document for continued work on the point source field visualizer. The README covers
what the thing is and how to use it; this covers where it stands, why it's built the way it
is, and what's unfinished.

---

## Current state

Working and complete:

- 16-partial additive editor, drag-to-draw amplitudes, per-partial phase
- Delay-lookup pressure field, 1 / 2 / 4 sources
- Separation in wavelengths; per-source phase offsets for B, C, D; four quad phase presets
- Signed diverging colormap, adjustable `1/rᵃ` display falloff
- Particle displacement overlay (single source only)
- Responsive layout: stacked below 900px, two-column with window-scaled field above
- Freeze/run

Two files carry the same behavior: `point-source-field.html` is the live one,
`point-source-field.jsx` is the earlier React version kept for reference. **The HTML is the
one to develop against.** If the React version stops mattering, delete it rather than
maintaining both.

### Code landmarks

Everything is one IIFE in a `<script>` at the bottom of the HTML.

| Symbol | Role |
| --- | --- |
| `S` | All mutable state. One plain object, read live by the render loop. |
| `rebuild()` | Recomputes `wave` and `disp` tables from `S.amps` / `S.phases`, then redraws the period trace and bars. Call this after any spectrum change. |
| `layout()` | Returns source positions as `{x, y, tag}` in normalized frame coords, `1.0` = half frame width. |
| `geometry()` | Builds one `Float32Array` distance field per source. Cached against `geoKey`. |
| `gains()` | Builds `gLUT`, the `1/rᵃ` lookup indexed by normalized radius. Cached against `gainKey`. |
| `frame()` | The render loop. Inner pixel loop sums one delayed table read per source. |
| `MAXPX` | Resolution ceiling per source count. |

The two caches matter: `geoKey` includes canvas size, source count, separation, and
wavelengths; `gainKey` is just falloff. Anything that changes those triggers a rebuild of
several megabytes of typed arrays, so don't add anything to those keys casually.

---

## Design decisions worth not re-litigating

**No wave solver.** Air is non-dispersive, so `p = g(r)·s(t − r/c)` is exact for a free field,
not an approximation. Anyone arriving fresh will be tempted to reach for FDTD. Don't — it
would be slower, less accurate here, and would lose the property that arbitrary spectra cost
nothing.

**Falloff is a display control, not physics.** Physically correct attenuation renders as a
bright dot surrounded by nothing. The exponent is exposed precisely so the honest value is
reachable but not mandatory.

**Signed colormap.** Absolute-value or magnitude coloring destroys the compression /
rarefaction asymmetry, which is the whole point — it's what makes odd versus even harmonic
content visible at a glance.

**Separation in wavelengths.** Pixel-denominated separation becomes meaningless the moment
zoom changes. Wavelength units keep the interference geometry stable.

**Source A is the phase reference.** Offsets read as relative delays. An absolute phase per
source would be four sliders where three carry the same information.

---

## Open threads

Roughly in order of value-to-effort.

### 1. Multi-source particle displacement

Currently disabled for N > 1. The overlay assumes a single radial center, so the dot offset is
a scalar along one direction.

Doing it properly: displacement is a vector, so each dot sums the per-source contributions,
each directed along that source's radial. For dot at position **p**:

```
ξ(p) = Σ_s  disp[t − r_s/c] · g(r_s) · (p − s_s)/|p − s_s|
```

Cost is N table reads plus a normalize per dot, roughly 3000 dots — cheap. The reason to do it
is that **velocity nulls and pressure nulls do not coincide**, and showing both simultaneously
in a four-source field is something a pressure-only plot can't communicate. This is the most
interesting unfinished item.

Once the dots are vector-driven, drop the polar grid in favor of a Cartesian one — the polar
arrangement only made sense when there was a single center.

### 2. Per-source spectra

All sources currently share one waveform table. Giving each its own means:

- `wave` / `disp` become arrays of tables, one per source
- `rebuild()` runs per source
- the inner loop indexes `wave[s]` instead of `wave`
- the UI needs a source selector above the partial editor, with the bars editing the selected
  source's spectrum

This is what turns it from a physics demo into a routing tool: four different spectra at four
positions is a multichannel output configuration, and the field shows exactly where the
partials build comb structure. Worth doing after item 1.

### 3. Prefiltered waveform table

The 3-tap box filter in the inner loop is the cheapest thing that suppresses aliasing on the
upper partials. It's adequate at the current 16 partials and zoom range, and it will fall apart
if either is pushed.

Proper fix: build a mip pyramid over the waveform table at `rebuild()` time — successive
half-resolution copies — and select a level from `perR / (W/2)`, the samples-per-pixel figure
already computed each frame as `h`. Trilinear between levels if the level transitions show.
Cost is a one-time build; the inner loop gets *cheaper* because it becomes one tap instead of
three.

### 4. Harmonic explode mode

Render each partial as its own ring layer before summing. Harmonic *n* has 1/n ring spacing,
so the nested-scale structure becomes visible, and dragging a partial's phase visibly rotates
its rings against the others. Discussed early, never built. Probably a small-multiples grid
rather than an overlay.

### 5. GLSL port

The whole inner loop is a fragment shader with no branching:

```glsl
// r from a varying or computed from source uniforms
float idx = fract(t - r * cycles);
float s = texture2D(waveTex, vec2(idx, 0.0)).r;
```

A `jit.gl.pix` or `jit.gl.slab` version would run at full frame resolution with all four
sources and no resolution ceiling, and would drop into a Max patch alongside the existing
projector-controller and light-show work. The waveform table becomes a 1D texture rebuilt from
`buffer~` or a `jit.matrix`. Source positions and offsets become uniforms.

The CPU version stays useful as the standalone/documentation artifact, but the shader is where
this wants to live if it becomes a performance instrument.

### 6. Real signal input

Replace the additive table with a captured buffer — one period of an actual 3rd Wave output,
or any recorded waveform. The renderer doesn't care where the table comes from; only
`rebuild()` changes. This is a small change with a large payoff for the wavetable-specific
case, since wavetable interpolation artifacts would be directly visible in the ring structure.

---

## Known limits

Not bugs — consequences of the model. Documented so they don't get "fixed" by accident.

- **Free field only.** No boundaries, reflections, absorption, or room. The field assumes the
  source has been running forever; there is no onset transient.
- **No sound engine.** Oscillation rate is nominal; frequency is normalized, not in Hz.
  Adding audio would mean deciding what the visual `c` corresponds to physically, which is
  currently a free parameter chosen for legibility.
- **2D cross-section.** A true 3D point source falls off as `1/r`; the exponent control lets
  you set that explicitly, but the rendered slice is not a cylindrical source.
- **Sources can leave the frame** at large separation with low wavelength counts. Deliberately
  unclamped — clamping would make the λ readout lie. Raising `Wavelengths` brings them back.

### Things to watch

- Four sources at a large window is the worst case: `pixels × 4` table reads per frame. The
  `MAXPX` ceiling handles it, but any added per-pixel work multiplies by source count.
- `geometry()` allocates fresh `Float32Array`s on every rebuild. At 1000px × 4 sources that's
  16MB churned per geometry change. Reusing the buffers when only positions change would be
  worth doing if source count ever becomes dynamic.
- The `resize` listener resets `geoKey` and redraws the period trace. Anything else that
  depends on layout needs adding there.
