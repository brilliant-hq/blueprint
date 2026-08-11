---
assumes: blueprint/paint
dsl: [uv, frozen]
---
# Shaders Overview

Six shaders, each a physical surface or pattern rather than an abstract
effect.

| Shader | Looks like |
|--------|-----------|
| `metaballs()` | organic blobs merging and parting, like a lava lamp |
| `metal()` | chrome with light-catching ridges and chromatic aberration |
| `irid()` | deep iridescent liquid crystal, shifting color bands |
| `steel()` | flowing mercury, smooth and reflective |
| `dithering()` | animated ordered-dither field, retro two-tone stipple |
| `reactiveGrid()` | glowing grid that warps toward the cursor when hovered |

## Never put text on a shader

Shader patterns make text unreadable. Keep them apart: put the shader on
a **stroke** (solid fill for the text), or as a **background** with a
semi-transparent `solid()` stacked over it as a second fill, text in a
separate child. A shader background is a fill, never a child: put the
shader fill on the frame itself, never a separate
`r s(fill,fill) f[(shader(...))]`.

## Controls

`scale(N)` zoom, `uvx(N)`/`uvy(N)` pan, `uvrot(N)` rotate, `opacity(N)`,
`speed(N)` animation rate, `frozen` to stop animation,
`shape(element|metaballs|none)` (metal, irid, steel): `element` fills the
layer's own silhouette (the default), `metaballs` renders drifting blobs,
`none` fills the whole quad. Numeric `shape(0|1|2)` is equivalent
(`0`=element, `1`=metaballs, `2`=none). On small surfaces like button
strokes, `scale(2-4)` shows more detail than `scale(1)`. Keep `speed` slow
on backgrounds, faster on strokes.

`dithering()` adds `size(N)` (dot scale) and numeric `shape(0-6)` /
`ditherType(0-3)`. `reactiveGrid()` adds `density(N)`, `distortion(N)`,
`radius(N)`; it warps toward the cursor in the live app only.

## Colors and strokes

Shader colors are tokens like any other slot: brand stops for cohesive
accents, Tailwind stops for fixed meaning.

```
f[(metaballs($primary.subtle,$primary.mid,$primary.strong))]
f[(irid($pink.mid,$violet.mid,$indigo.mid))]
st[(steel($amber.mid,$amber.faint),w($stroke.width.mid))]
```

Strokes take every fill type, shaders included: a `metal()` chrome edge
or `irid()` shimmer border is one of the most underused looks. Metaballs
`size`/`count` density: see `blueprint/shaders/metaballs`.
