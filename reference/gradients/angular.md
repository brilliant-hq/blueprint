---
assumes: blueprint/paint
dsl: [angular, conic, cx, cy, ax, ay]
---
# Gradients: Angular

Assumes: `blueprint/paint`

Colors sweep around a center point (conic/sweep). Like a color wheel or light refracting through a prism.

## Syntax

```
(angular())                                    — default: centered, black→white
(angular(#start,#end))                         — two-color
(angular(cx,cy,ax,ay,stop(#hex,pos),...))       — full control
(angular(cx,cy,ax,ay,w(wx,wy),stop(#hex,pos),...))  — elliptical
```

Token-bound stops work the same as in linear/radial:
`angular($brand.mid,$brand.intense)`, `stop($neutral.hint,0,o($visibility.mid))`,
`solid($brand.mid,o(0.5))`.

## Use Cases

**Progress rings** — angular gradients naturally follow circular paths:
```
st[(angular($amber.mid,$red.mid),w($stroke.width.bold),cap(r,r))] arc(90,84) ratio(1)
```

**Metallic/holographic sheen** — close hues create brushed-metal look:
```
f[(angular(stop($neutral.soft,0),stop($neutral.faint,0.25),stop($neutral.soft,0.5),stop($neutral.subtle,0.75),stop($neutral.soft,1)))]
```

**Badge accents** — small angular on circular elements creates gem-like refraction:
```
al(h,x(c),y(c),g($spacing.none),pad($spacing.none)) s(32,32) f[(angular($emerald.mid,$teal.mid,$cyan.firm,$emerald.mid))] rd($radius.full) "Badge" #badge
```

## When to Reach for Angular

- Progress rings and gauges
- Metallic/holographic sheen on circular shapes
- Color wheel / spectrum displays
- Badge and icon accents on circular elements
