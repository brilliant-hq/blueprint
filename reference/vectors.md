---
assumes: blueprint/paint
dsl: [v(, mi, st, as, di]
---
# Blueprint Vectors

Assumes: `blueprint/core`, `blueprint/paint`

## Syntax

`v(nodes[(id,x,y,type),...],edges[(id,nodeA,nodeB,haX,haY,hbX,hbY),...],closed)`

**Node types:** `st` (straight, default) · `mi` (mirrored/smooth) · `as` (asymmetric) · `di` (disconnected)

**Auto-smooth curves:** Mark nodes as `mi`, system computes tangent-based handles at 30% of edge length. No manual handle coordinates needed. Edges without explicit handles default to `(0,0,0,0)` and are auto-computed for `mi` nodes.

**Auto edges:** If edges are omitted entirely, they're generated sequentially (node 0→1→2→...).

## Stroke-only curve

```
v(nodes[(0,0,40,mi),(1,60,16,mi),(2,120,12,mi)]) s(120,48) st[($teal.mid,w($stroke.width.soft))]
```

## Area fill (closed path)

Close with straight bottom nodes + outside stroke + clip frame to hide closing edges:
```
al(v,g($spacing.none),pad($spacing.xs,$spacing.none,$spacing.none,$spacing.none)) s(hug,hug) clip "ClipFrame"
  v(nodes[(0,0,40,mi),(1,60,16,mi),(2,120,12,mi),(3,120,48),(4,0,48)],edges[(0,0,1),(1,1,2),(2,2,3),(3,3,4),(4,4,0)],closed) s(120,48) f[($teal.faint)] st[($teal.mid,w($stroke.width.soft),pos(o))]
```

Outside stroke `pos(o)` pushes boundary strokes beyond the bbox. Clip frame crops them. Top padding >= stroke width. ONE vector with both fill and stroke.

## Coordinates

`(0,0)` = top-left (highest value), `(W,H)` = bottom-right (lowest). Every node MUST have 3 to 5 values: `(index,x,y)`, `(index,x,y,type)`, or `(index,x,y,type,cap)` (the 5th stroke-cap slot is covered under Full-fidelity networks below).

## Full-fidelity networks

With explicit edges, arbitrary topology is preserved exactly, shared
nodes, T-junctions, dangling stroke edges. A node takes an optional 5th
slot for its stroke cap: `(0,0,40,mi,ar)` (`r`/`n`/`sq`/`ar`/`c`).
Regions (faces) can be declared explicitly inside `v()`:
`regions[(0+1+2+),(3+4-5+,hole)]`: each entry walks its boundary edges
(`edgeId` + `+`/`-` direction), `hole` subtracts. Assign a region's fill
with a `vr(rN) f[...]` continuation line (rN = 1-based position in the
regions list). Reading a canvas returns vectors in this same form.
