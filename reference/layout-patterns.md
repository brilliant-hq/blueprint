---
assumes: blueprint/layout
dsl: [abs, wrap, front, back, cb]
---
# Blueprint Layout Patterns

## Common patterns
```
al(h,x(sb),y(c),g($spacing.md),pad($spacing.none)) s(fill,hug)   ← push 2 children to opposite edges (spaceBetween)
al(h,x(e),y(c),g($spacing.md),pad($spacing.none)) s(fill,hug)   ← push all children to the right
al(h,x(c),y(c),g($spacing.sm),pad($spacing.none)) s(hug,hug)    ← self-sizing centered row (button, chip)
al(v,y(sb),x(s),g($spacing.none),pad($spacing.none)) s(fill,600)           ← pin footer to bottom (g($spacing.none) = intentional zero gap)
al(h,g($spacing.overlap.md),pad($spacing.none))              ← negative gap pulls children into overlap (avatar stacks, badge over icon, hero break-out). Scale: $spacing.overlap.xs/sm/md/lg/xl/2xl (-2 → -32)
al(h,g($spacing.sm),pad($spacing.none)) s(fill,hug)          ← N equal-width items: children s(fill,fill); fixed+flexible: s(240,fill) + s(fill,fill)
al(v,y(e),x(c),g($spacing.sm),pad($spacing.none)) s(hug,fill)   ← bottom-align in column (container needs fixed height)
gr s(200,200)                              ← group for free p(x,y) positioning; p(c,c) centers a child
  r abs p(c,c) s(100,100)                 ← abs: free positioning inside auto layout
```

## Alignment: `x(...)` / `y(...)`
Axes are physical, never flip with direction (`x`=horizontal, `y`=vertical). `sb` is main-axis only (`x(sb)` in `al(h)`, `y(sb)` in `al(v)`) and needs a non-`hug` parent with room to distribute. Bottom-center column → `y(e),x(c)`; right-center row → `x(e),y(c)`. Never make empty spacer frames, items right → `x(e)`, opposite edges → `x(sb)` (2 children), top+bottom → `y(sb)` (2 children, fixed/fill height).

## Grid patterns
Row-aligned (table): `al(v,g(1))` → `al(h,g(0)) s(fill,hug)` rows → `s(fill,fill)` cells.
Column-major (kanban): `al(h,g($spacing.md))` → `al(v,g($spacing.sm)) s(fill,fill)` columns.

## Group vs Frame vs Absolute
Frame/auto-layout: structured content (gap, padding, alignment) + linear overlap via `g($spacing.overlap.*)`. Group: free-form `p(x,y)` positioning, charts, decorative/rotated layers. `abs` on an auto-layout child: ignores flow, positions with `p(x,y)`, stays a child (clips/moves/undoes with parent). `p(c,c)` centers in any parent. Pick: badges/dots/floating buttons → `abs`; chart overlays (arcs, rings) → `gr`; avatar/card stacks, badge biting icon → `al` + `g($spacing.overlap.*)`.

⚠ **`al(...)` with `x(c)/y(c)` does NOT stack children on top of each other**, centering is within flow, children still run end-to-end. **Concentric** overlap (shared center): `gr` + `p(c,c)`, or `al` + `abs p(c,c)` on every child. **Linear** overlap (siblings pulled together): `al(h,g($spacing.overlap.md))`.

## Radial placement (petals, ticks, orbiting dots)
N copies on a circle: **compute each `p()`/`rot()`, never eyeball** (estimates come out lopsided). For copy at angle `θ` (0=up, step `360/N`): `p.x = cx + R·sinθ − halfW`, `p.y = cy − R·cosθ − halfH`. `cx,cy`=hub (group center), `R`=orbit radius, `halfW/H`=half the copy's *rendered* size (subtract: `p()` is top-left). 8 petals, hub (140,140), R=58, copy 64×130:
```
gr p(c,c) s(280,280) "Lotus"
  v(...) comp p(108,17) s(80,130) f[(...)] "Petal" #petal  -- θ=0 top. comp on a vector wraps it; rendered size (here 64×130) may differ from s(), use it for halfW/H
  inst(#petal) p(149,34)  rot(45)   -- rot() pivots on the copy's OWN center, so position FIRST then rotate, rotating in place collapses all copies onto the hub as a star
  inst(#petal) p(166,75)  rot(90)   -- right
  inst(#petal) p(149,116) rot(135)
  inst(#petal) p(108,133) rot(180)  -- bottom
  inst(#petal) p(67,116)  rot(225)
  inst(#petal) p(50,75)   rot(270)  -- left
  inst(#petal) p(67,34)   rot(315)
  c p(c,c) s(64,64) f[(...)] "Hub"  -- center last = on top
```

## Debugging
Empty fill child invisible → no content to hug-fallback to; give it children or a fixed size. spaceBetween not working → parent is `hug`, needs concrete size. `p(x,y)` no effect → in auto-layout, add `abs`. Text wrapping unexpectedly → text has `fill`, change to `hug`. Bottom not pinned → parent `hug` height, needs fixed/fill. Items jammed → parent has `g(0)`. Common sizes: iPhone 393×852 · iPad 1024×1366 · Laptop 1440×900 · Desktop 1920×1080.
