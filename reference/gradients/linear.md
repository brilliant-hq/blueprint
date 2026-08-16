---
assumes: blueprint/paint
dsl: [linear, angle, multi-stop, shear, skew]
---
# Gradients: Linear

A directional sweep across a surface. `linear(angle, stops...)`: angle in
degrees, then two or more color stops; with no angle it defaults to 180.
Stops are tokens like any color slot, optionally with a position and
opacity.

```
f[(linear(180,$primary.hint,$primary.intense))]                          two bound endpoints
f[(linear(135,stop($neutral.hint,0),stop($neutral.intense,1)))]           explicit positions
f[(linear(180,solid($primary.mid,o($visibility.mid)),$primary.intense))]  per-stop opacity
```

**Angle**: `180` top to bottom, `0` bottom to top, `90` left to right,
`135` the energizing diagonal. Token-bound stops follow brand and mode
switches, same as a solid fill.

## Shear (third handle)

Add `w(wx,wy)` to skew the color bands off perpendicular, so the iso-lines
run at an angle to the sweep direction instead of square across it. `wx,wy`
place the shear handle in the same `0..1` space as linear's positional
coords. Leave `w()` off for a plain perpendicular linear (the default,
unchanged):

```
f[(linear(135,w(1,0),$violet.mid,$pink.mid))]
```

Reach for it to match an imported gradient whose bands are not square to its
direction, or for a subtle skew on a large surface. Most linears want none.

## Multi-stop

Two stops read flat; a third adds depth. The middle stop is the "knee"
where the gradient visibly bends, so give it a color that shifts:

```
f[(linear(180,stop($amber.soft,0),stop($pink.mid,0.6),stop($violet.bold,1)))]
```

To make an end color read as a true accent rather than a continuation,
hold the first color across the first half:

```
f[(linear(135,stop($rose.bold,0),stop($rose.bold,0.5),stop($amber.soft,1)))]
```

Gradients are the cheapest way to add warmth and depth; a
neutral-on-neutral gradient is usually a missed opportunity.

## Per-stop opacity

Each stop can carry `o(N)` or `o($visibility.*)`. The classic
text-on-image overlay is an image base plus a gradient fading from
transparent at the top to near-opaque at the bottom:

```
fr s(W,H) clip f[(img(...)),(f2,linear(180,solid($neutral.intense,o($visibility.invisible)),solid($neutral.intense,o($visibility.strong))))]
```
