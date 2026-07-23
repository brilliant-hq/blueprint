---
assumes: blueprint/paint
dsl: [radial, cx, cy, rx, ry, elliptical]
---
# Gradients: Radial

A point-source light: spotlights, glowing orbs, focal gravity that a
linear gradient cannot give.

## Syntax

```
radial($center,$edge)                        two-color, centered
radial(cx(25),cy(15),r(50),$c1,$c2)           positioned (0 = left/top, 50 = center; r: 50 = edge)
radial(rx(80),ry(40),$c1,$c2)                 elliptical
radial(cx,cy,rx,ry,stop($c,pos),...)          positional, multi-stop
```

Stops take tokens for color and opacity, like any fill. Bind to the
brand (`$primary.mid`) so the gradient follows mode and brand switches;
a raw hex would not.

## Patterns

Ambient glow: off-center, low opacity, brand-tinted; layer two or three
for a mesh-gradient feel.

```
f[(radial(-0.5,-0.5,0.5,0.5,stop($primary.bold,0,o($visibility.soft)),stop($primary.bold,1,o($visibility.invisible))))]
```

Vignette: transparent center, dark edges (`$color.shadow` stays dark in
both modes).

```
f[(radial(0,0,1,1,stop($color.shadow,0,o($visibility.invisible)),stop($color.shadow,0.6,o($visibility.invisible)),stop($color.shadow,1,o($visibility.bold))))]
```

Radial over linear: the linear sets the mood, a radial adds a highlight.

```
f[(linear(135,$zinc.intense,$indigo.intense)),(f2,radial(0.3,-0.3,0,-1,stop($indigo.mid,0,o($visibility.subtle)),stop($indigo.mid,1,o($visibility.invisible))))]
```

## Instincts

Off-center by default (centering every radial is an amateur tell).
Prefer elliptical over circular, since real light pools stretch. Layer
two faint radials rather than one strong one.
