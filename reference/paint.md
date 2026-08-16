---
assumes: blueprint/core
dsl: [solid, metaballs, metal, irid, steel, dithering, reactiveGrid, glass, colorAdjust, noiseGrain, halftone, pixelate, duotone, posterize, dither]
---
# Blueprint Paint

`f[(...)]` is a compositing stack: list multiple fills and they render in
order (tints, gradients, blurs, glows, shaders, dim overlays are all fill
layers). Never make a separate child rectangle for a background or
overlay; stack it as another fill on the frame itself.

## Fills

```
f[($color.primary)]                                 semantic role, mode-flips
f[(solid($color.secondary,o($visibility.mid)))]     role + opacity
f[(linear(135,$violet.mid,$pink.mid))]              linear gradient
f[(radial($color.surface,$color.on-surface))]       radial gradient
f[(angular($amber.mid,$red.mid))]                   angular gradient
f[(diamond($amber.mid,$red.mid))]                   diamond gradient (four-point rhombus)
f[(linear(135,w(1,0),$violet.mid,$pink.mid))]       linear + shear (third handle)
f[(metaballs($primary.soft,$primary.mid,$primary.bold))]  shader
f[(dithering($neutral.intense,$cyan.mid,size(6)))]  shader (ordered dither)
f[(reactiveGrid())]                                 shader (grid, cursor-reactive)
f[(img(https://picsum.photos/id/42/800/400))]       image
f[(img(...)),(halftone(dotSize(8),mode(cmyk)))]     image filter over the fill below
f[(glass)]                                          liquid glass (defaults)
f[(glass(chroma(0.5),frost(8)))]                    glass, params overridden
f[($neutral.intense),(f2,solid($primary.mid,o($visibility.soft)))]  stacked
f[]                                                 deliberately no fill (outline-only text/shapes; keeps a stroke visible)
```

Liquid glass refracts the backdrop like a glass pane. Bare `glass` uses the
tuned defaults; override any of `frost(N)` (backdrop blur), `thickness(N)`,
`bevel(N)`, `ior(N)`, `chroma(N)`, `glow(N)`, `edge(N)`, `angle(deg)`,
`sat(N)`, `tint(color[,o(N)])`, a `$token` or hex tint. Fill-only (not a
stroke).

Every color slot takes a `$token` (solid fills, gradient stops, shader
colors, effect colors); token-bound stops follow brand and mode. While a
design system is active, bare hex in these slots is rejected: use a `$token`
(see [`design-systems/core`](https://github.com/brilliant-hq/brilliant/blob/main/knowledge/design-systems/core.md)). With no design system, use hex directly (a `$token` is what
gets rejected then). Radial accepts placement:
`radial(cx(25),cy(15),r(50),$primary.mid,$primary.intense)`. Diamond shares
radial's geometry (`diamond(cx,cy,ex,ey,...)` or the shorthand above; it has no
`cx()/cy()/r()` percentage form). Linear takes an optional `w(wx,wy)` third
handle to shear/skew the sweep; leave it off for a plain perpendicular gradient.
Full per-kind guidance lives in `blueprint/gradients/*`.

## Image filters

Image filters restyle whatever renders beneath them in the stack, so put one
after the fill it acts on: `f[(img(...)),(halftone(mode(cmyk)))]`. Bare uses
tuned defaults; pass named params as numbers or option labels (`shape(hexagonal)`).
`opacity(N)` sets the filter's own strength.

```
colorAdjust  photo grade: exposure, contrast, saturation, temperature, hue, vibrance, whites, blacks, vignette
noiseGrain   film grain: amount, size, monochrome(color|mono)
halftone     dot screen: dotSize, angle, shape(circle|diamond|line), mode(standard|cmyk)  (colors: bg, fg, +CMYK inks)
pixelate     mosaic: cellSize, shape(square|hexagonal|diamond|circle|triangle)
duotone      two-tone map (2 color slots): contrast, brightness, saturation
posterize    banding: levels, mode(rgb|luminosity|hsl)
dither       ordered dither (2 color slots): levels, pattern(bayer|noise|blueNoise), pixelSize
```

**Editing one fill (`+`/`-`/`->`).** When modifying an element, a bare `f[...]`
replaces the whole stack; prefix items to edit it in place instead. Match is by
paint: `#card f[+($color.primary)]` appends, `f[-($color.primary)]` removes that
fill, `f[($color.primary)->($color.danger)]` swaps it. Same operators on `st[]`.

## Strokes

`st[(paint,w(N)),...]` takes every fill type (solids, gradients, shaders)
with token stops, exactly like `f[]`.

```
st[($color.outline.variant,w($stroke.width.subtle))]
st[(linear(90,$pink.mid,$amber.mid),w($stroke.width.mid))]
st[(metal($amber.mid,$amber.faint),w($stroke.width.mid))]
```

`pos(c|i|o)` aligns the stroke center / inside / outside. Caps are set
per node (vector endpoints) and per circle, not on the stroke.

`w(t,r,b,l)` sets per-side widths on rectangles and frames. With a design
system active, give every side a stroke-width token; `$stroke.width.none` is
no border on that side. A bottom divider:
`st[($color.outline.variant,w($stroke.width.none,$stroke.width.none,$stroke.width.subtle,$stroke.width.none))]`.
In bare-value mode (no design system) use plain numbers instead: `w(0,0,1,0)`.
Don't mix bare numbers and tokens in one `w()`: a bare number is rejected while
a design system is active, so keep all four sides tokens.

## SVG icons & region fills

`svg(icon:house)` bundled Phosphor (kebab-case), `svg(https://...)` URL,
`svg(/path)` local, `svg(<svg>...)` inline. A fill on the `svg` line
overrides imported fills (creation-only; recolor later with the
`recolor_children` command). On a vector with multiple closed regions,
`vr(rN) f[...]` fills one region by ID.
