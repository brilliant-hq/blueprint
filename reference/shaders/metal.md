---
assumes: blueprint/shaders/overview
dsl: [metal, chrome, softness, repetition, aberration]
---
# Shader: Liquid Metal

`metal()` is chrome or brushed aluminum with chromatic aberration: a
material with weight that catches and bends light. Default `metal()` is
silver chrome.

## Parameters

`softness(N)` edge smoothness, `repetition(N)` stripe count (more reads
as brushed texture), `contour(N)` ridge visibility, `angle(N)` light
direction, `shiftRed(N)` / `shiftBlue(N)` chromatic aberration,
`speed(N)` animation. Colors are positional token args.

## Colors

Shader colors are tokens. Chrome and metal tones come from Tailwind
neutrals (`$zinc.*`, `$slate.*`, `$gray.*`). For a tuned recipe (gold,
copper, rose chrome) that no Tailwind stop matches, extend the active DS
with `art.*` primitive tokens and reference them as `$art.<name>`.

## Examples

A metal fill is busy, so metal works best on a thin accent. A metallic
divider:

```
r s(200,3) f[(metal($amber.strong,$amber.hint,softness(0.5),speed(0.2)))] rd($radius.full) "Divider"
```

For a button, put `metal()` on the stroke and keep the text on a solid
fill: `f[($color.surface)] st[(metal(),w($stroke.width.mid))]`. On small
surfaces, `scale(2-4)` shows more of the pattern.

Glass over a metal background: see `effects/glass`.

## Best uses

Use metal on strokes, accents, and dividers, never as a fill under text.
The chromatic aberration reads best on a thin edge.
