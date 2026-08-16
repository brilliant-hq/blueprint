---
assumes: blueprint/paint
dsl: [diamond, rhombus, gem, cx, cy, rx, ry]
---
# Gradients: Diamond

Assumes: `blueprint/paint`

Colors radiate along a four-point rhombus (the diamond isoline, `|x|+|y|`)
instead of a circle. Reads like a faceted gem, a crystal, or a cross/star
highlight pooling toward a focal point. The fourth gradient kind, sharing the
family's three-handle geometry with linear, radial, and angular.

## Syntax

```
(diamond()), default: centered, black→white
(diamond($start,$end)), two-color
(diamond(cx,cy,ex,ey,stop($c,pos),...)), positional, multi-stop (ex,ey = end point)
(diamond(cx,cy,ex,ey,w(wx,wy),stop($c,pos),...)), skewed / stretched rhombus
```

Diamond stores exactly like radial: the bare positional form is alignment
space (`0` = center, `±1` = the edges), and the 3rd/4th values are the
gradient's **end point** (which sets both radius and direction), not `rx`/`ry`
radii. The optional `w(wx,wy)` is the secondary axis, so the rhombus can
stretch or skew off-square. There is no `cx()/cy()/r()` percentage form (that
is radial-only): use the positional form or the shorthand.

Token-bound stops work the same as linear/radial/angular:
`diamond($brand.mid,$brand.intense)`, `stop($neutral.hint,0,o($visibility.mid))`,
`solid($brand.mid,o(0.5))`.

## Use Cases

**Faceted gem / crystal**: close hues around a center read as cut facets:
```
f[(diamond(stop($violet.mid,0),stop($fuchsia.mid,0.5),stop($violet.mid,1)))]
```

**Cross / star highlight**: a bright center falling off along the diamond
axes gives a sparkle a radial cannot:
```
f[(diamond($amber.hint,$amber.mid))]
```

**Gem badge stroke**: a diamond gradient on a border for a jewel edge:
```
st[(diamond($amber.mid,$red.mid),w($stroke.width.bold))]
```

## When to Reach for Diamond

- Faceted gem, crystal, and jewel fills
- Cross/star or sparkle highlights on a focal point
- Angular, hard-edged glows where a radial feels too soft
- Reflective/metallic accents on square or rhombus shapes
