---
assumes: blueprint/core, blueprint/components
dsl: [arc, ring, sweep, cap, ratio]
---
# Blueprint Arcs & Rings

For any circular progress, ring, or pie, use a `c` with `arc()` + `ratio()`.
Never hand-roll arcs as `v()` paths. (To *distribute* N copies around a
circle, see radial placement in `blueprint/layout-patterns`.)

`arc(start,sweep)`: `start` in degrees (0 = 3 o'clock, 90 = top, so
progress rings start at `arc(90,...)`), `sweep` a PERCENTAGE (0-100) of a
full circle filled clockwise, NOT degrees. So `arc(90,75)` is a
three-quarter arc, `arc(90,50)` a half. ⚠ A sweep above 100 is a hard error:
the tool refuses the line and points you to the percentage form rather than
silently shipping a full circle. The classic slip is thinking in degrees,
e.g. `arc(0,270)` for a 3/4 arc; write `arc(0,75)` instead (`arc(0,100)` is a
full circle). Opening an older file that stored a >100 sweep still loads: it
just reads as a full circle.
`ratio(N)`: inner radius as a fraction of outer. Omit it for a solid
circle, `ratio(1)` for a stroke-only ring. Concentric rings go in a `gr`
with `p(c,c)` on each child; an `al` would stack them in flow.

## Progress ring

A track circle and a same-size arc circle, plus center text:

```
gr s(80,80)
  c s(80,80) st[($color.outline.variant,w($stroke.width.bold))] ratio(1) "Track"
  c s(80,80) st[($color.success,w($stroke.width.bold),cap(r,r))] arc(90,75) ratio(1) "Arc"
  t("75%",$font.family,$font.size.lg,b) p(c,c) f[($color.text.primary)] "Pct"
```

`cap(r,r)` rounds the arc ends below ~80% sweep; omit caps above ~90%
(round caps overlap at full sweep). Completed ring: a full `arc(90,100)`
plus `outerglow($color.success,o($visibility.mid),blur(12))`. Multi-ring:
stack `c` pairs at shrinking sizes (80, 56, 36) in one `gr`, tracks at
`o($visibility.soft)` so they recede behind the colored arcs.

## Pie / donut

Same-size arc circles, each slice's start trailing the last:
`next_start = prev_start - prev_sweep% × 3.6`. Wrap a slice in a `gr` so
`comp`/`inst` is valid, then override the arc per instance (marking a
bare arc-circle `c` as `comp` now auto-promotes it into a wrapper frame,
so the explicit `gr` is optional but keeps the structure clear):

```
gr s(100,100)
  gr comp p(c,c) s(100,100) #slice
    c p(c,c) s(100,100) f[($blue.mid)] arc(90,45) ratio(0.55) #slice_arc
  inst(#slice)
    override(#slice_arc) c p(c,c) s(100,100) f[($emerald.mid)] arc(288,25) ratio(0.55)
```

Slice colors are distinct palette stops (`$blue.mid`, `$emerald.mid`,
`$amber.mid`, `$violet.mid`, `$rose.mid`), never semantic roles: data viz
wants fixed hues that don't mode-flip. Omit `ratio()` for a solid pie.
