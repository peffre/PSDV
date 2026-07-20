# Point Source — Compression / Rarefaction Field

An interactive visualization of acoustic pressure radiating from one, two, or four point
sources, driven by a 16-partial additive spectrum editor.

Compression is warm, rarefaction is cool, ambient pressure is the neutral background. The
concentric ring structure you see **is** the source waveform, drawn outward and reversed in
time — change the spectrum and the rings change shape immediately.

Single self-contained HTML file. No build step, no dependencies, no network access. Open
`point-source-field.html` in a browser, or point a `jweb` object at it to embed it in a Max
patch.

---

## Method

The visualization does not solve the wave equation. It doesn't need to.

Air is non-dispersive: every partial travels at the same speed `c`, so the waveform shape is
preserved as the wavefront expands. Pressure at any point is therefore just the source signal
read at a delay, scaled by distance:

```
p(x, y, t) = g(r) · s(t − r/c)        r = distance from the source
```

That reduces to one table lookup per pixel per source. Consequences worth knowing:

- **No stability constraint.** There's no CFL condition, no grid dispersion, no numerical
  damping. Nothing to tune for the solver's sake.
- **Arbitrary spectra are free.** Inharmonic partials, transients, and discontinuities all
  work, because the code reads an actual signal rather than summing a truncated Fourier
  series at render time.
- **Multiple sources are a sum.** Interference is exact, not emergent from a discretization.
- **It's a snapshot of steady state.** There are no boundaries, no reflections, and no
  onset — the field assumes the source has been running forever. Room acoustics are out of
  scope by construction.

The source waveform is built once into a 2048-sample table (`LUT`) whenever the spectrum
changes, along with its running integral, which gives particle displacement.

### Aliasing

At high `Wavelengths` settings, the upper partials approach one cycle per pixel and would
alias into moiré. Each lookup is a 3-tap box filter whose width tracks one pixel of radius
(`h` in the render loop), which keeps partial 16 clean across the full zoom range. It's the
cheapest thing that works; a proper mip pyramid over the waveform table would be better if
you push the partial count higher.

---

## Controls

### Harmonic partials

Drag across the bars to draw the spectrum — sixteen partials, amplitude only. The selected
partial's phase gets its own slider below.

Phase is worth playing with: sweep partial 3 or 5 and watch the ring *shape* change while the
magnitude spectrum stays identical. Presets are the usual set — Sine, Saw, Square, Triangle,
Pulse (all partials equal), Odd — plus Clear, which returns to a bare fundamental.

The **Period** trace to the right shows one cycle of the resulting waveform. Everything in the
field is derived from that trace.

### Sources

| Setting | Layout |
| --- | --- |
| One | Centered |
| Two | Horizontal pair, A left / B right |
| Four | Square, A/B on top, C/D below |

`Separation` is measured **in wavelengths**, not pixels, so the interference geometry stays
meaningful when you change the zoom. For four sources it's the side length of the square.

Source A is always the phase reference at 0°; B, C, and D get their own offset sliders, so the
numbers read as relative delays. The four-source quad presets:

- **In phase** — center antinode with a null grid around it.
- **Alternating** — diagonal pairs opposed. The center goes permanently null and energy
  squeezes out along the diagonals.
- **Rotating** — 0 / 90 / 270 / 180 around the square. The field circulates rather than
  standing. With a rich spectrum each partial appears to rotate at a different rate, because
  the offset is fixed in time rather than in cycles.
- **Pair flip** — front pair against rear pair; the dipole figure-eight.

The radiating dark spokes are comb nulls. Each partial nulls at a different angle, which is
the visual reason a comb filter sounds like a comb rather than a hole.

### Field

- **Wavelengths** — how many wavelengths span the radius of the frame. This is the zoom.
- **Falloff 1/rᵃ** — display gamma on the distance attenuation, not physics. `a = 2.0` is
  honest inverse-square and mostly unreadable: you get a bright dot and little else. Around
  0.5 keeps the outer wavefronts visible.
- **Particle displacement** — overlays a polar dot field driven by the integral of pressure.
  Dots bunch at the color *transitions*, not the color peaks, because displacement and
  pressure are 90° apart. Single-source only; the overlay assumes one radial center, and doing
  it properly for multiple sources means summing vector displacements per dot.
- **Freeze / Run** — stops time. Useful for reading fine ring structure or for screenshots.

---

## Layout and performance

Below 900px viewport width the interface is a single stacked column. At 900px and above it
switches to a two-column grid: the field scales to fill the window height, and the controls
move to a scrolling side rail. The page itself doesn't scroll on desktop, so the field stays
fixed while you work.

Render cost is roughly `pixels × sources`, so the internal resolution ceiling steps down as
sources are added:

```js
const MAXPX = {1: 1000, 2: 820, 4: 660};
```

Raise these if your machine has headroom — the four-source field is upsampled slightly by CSS
at large window sizes, which is barely visible given how smooth the field is.

Two things keep the loop fast: the `1/rᵃ` gain is a precomputed lookup indexed by normalized
radius rather than a `Math.pow` per tap, and the per-source distance fields are cached and
rebuilt only when the geometry actually changes. Moving the falloff or wavelength sliders
triggers no rebuild at all.

---

## Extending it

State lives in one plain object, `S`, and the render loop reads it live. A few entry points:

- **Drive the spectrum externally.** Write into `S.amps` / `S.phases` and call `rebuild()`.
  That's the whole contract — hook it to analysis data, MIDI, or OSC.
- **Add a fifth source.** `layout()` returns an array of `{x, y, tag}` in normalized frame
  coordinates where `1.0` is half the frame width. The renderer is generalized over that list;
  extend the array and add an entry to `S.off`.
- **Change the colormap.** `buildPalette()` builds a 512-entry diverging ramp. Keep it signed —
  taking the absolute value destroys the compression/rarefaction asymmetry that makes odd
  versus even harmonic content visible in the first place.
- **Per-source spectra.** Currently all sources share one waveform table. Giving each its own
  table means one `rebuild()` per source and indexing `wave` by source in the inner loop.

### Known limits

- No sound engine. Oscillation is a fixed nominal rate; frequency is normalized rather than
  in Hz.
- Free field only — no boundaries, reflections, or absorption.
- The 2D slice is a cross-section, not a cylindrical source. Real point-source falloff in
  three dimensions is `1/r`, which the falloff exponent lets you set explicitly.
- Particle displacement is single-source only.

---

## Files

```
point-source-field.html   the whole thing
point-source-field.jsx    earlier React version, same behavior
README.md
```
