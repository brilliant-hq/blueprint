---
assumes: blueprint/core, blueprint/paint
dsl: [line, len, angle, from, to, route, avoid, intent]
---
# Blueprint Lines

`line(...)` is the only way to draw a straight line in the DSL. It's
sugar over a 2-node vector, the resulting element is still a `vector`
under the hood, so caps, strokes, and gradients work exactly like other
vectors. Two argument shapes cover every use case:

```
line(len(N))                                vector form, horizontal len
line(len(N), angle(deg))                    vector form, len + direction
line(from(x,y), to(x,y))                    endpoint form, literal points
line(from(#a), to(#b))                      endpoint form, element refs
line(from(20,30), to(#b))                   mixed, coord to element
```

`connect(...)` was removed. Use `line(from(#a), to(#b))` instead; every
option (route, avoid, intent, anchors, nested-child anchors) works on
endpoint-form lines.

## Decision matrix

| You want… | Form |
|-----------|------|
| 1px divider in a layout | `r s(fill,1) f[...]`: not a line at all |
| Horizontal segment of length N | `line(len(N))` |
| Tilted segment of length N at angle | `line(len(N), angle(deg))` |
| Line between two coordinates | `line(from(x,y), to(x,y))` |
| Arrow between two elements | `line(from(#a), to(#b)) cap(n,ar) st[...]` |
| Arrow with smart routing | `line(from(#a), to(#b), intent(dependency))` |
| Polyline / curve / closed path | `v(nodes[...])`: not a line |

Lines always render with a stroke (open paths can't be filled). Add
`st[...]` or use `intent(...)` for a preset stroke. The parser rejects
strokeless lines at parse time.

## Vector form: `line(len(N) [, angle(deg)])`

```
line(len(120)) st[($color.outline,w(1))]                       horizontal
line(len(120), angle(45)) st[($color.outline,w(1))]            tilted
line(len(120), angle(180)) st[($color.outline,w(1))]           leftward
```

`angle(deg)` pivots the line at its **start point** (the `p()` value, or
0,0 if omitted). The element's `aabb` covers the rotated extents, so a
180° line places its visual end to the left of `p()`.

Negative lengths are allowed: `line(len(-100))` is identical to
`line(len(100), angle(180))`. Pick whichever reads clearer.

## Endpoint form: `line(from(...), to(...))`

```
line(from(0,0), to(120,40)) st[($color.outline,w(1))]          coord → coord
line(from(#a), to(#b)) st[($color.outline,w(1))] cap(n,ar)     ref → ref
line(from(#card), to(200,40)) st[($color.outline,w(1))]        mixed
line(from(#a), to(#b, r, 0.25)) intent(dependency)             anchor 25% down b's right edge
line(from(#card_port), to(#b)) intent(dependency)              nested-child anchor, resolved on the routing parent
```

Endpoint shapes per side:

```
from(x, y)            literal coord (parent-local)
from(#ref)            anchor on an element, auto-pick the closest side
from(#ref, side)      anchor on a named side (tl t tr l c r bl b br)
from(#ref, side, f)   anchor on a side at fractional position f (0..1)
from(#child_ref)      anchor on a nested element of the routing parent
```

With **both** endpoints as `#ref`s, the line auto-reparents to the
**lowest common ancestor** of its endpoints, so its z-order and clipping
follow the elements it joins. (Auto-reparent is skipped inside a
component master, and when either endpoint is a coordinate there is no
element pair to reparent to.) Inside an auto-layout parent it is
auto-`abs` so it doesn't shift siblings. Override placement with
`parent(#frame)`.

With a coordinate endpoint, the line stays at the position you gave it.

## Routing, avoidance, intent: endpoint form modifiers

These work on any endpoint-form line (coord or ref endpoints).

```
route(straight|elbow|elbow2|bezier)      path shape, default elbow
avoid(none|endpoints|all)                obstacle dodging; default all with two ref endpoints, none when either endpoint is a coordinate
intent(dependency|flow|annotation)       preset route + avoid + cap + stroke
```

- `intent(dependency)`: elbow + arrow, thin gray. Prerequisite chains, Gantt links.
- `intent(flow)`: elbow + arrow, bold dark. Flowchart and sequence steps.
- `intent(annotation)`: straight, no arrow, hairline gray. Callouts.

Intent presets seed defaults; an explicit `route()`/`avoid()`/`st[]`/
`cap()` always wins. Pass `intent()` whenever you don't need to fight the
defaults, it's the fastest path to a polished result.

```
line(from(#a), to(#b), intent(flow))                       preset
line(from(#a), to(#b), intent(flow), route(straight))      override one piece
line(from(#a), to(#b), route(elbow), avoid(all)) st[...]   explicit
line(from(20,30), to(#b), route(elbow), avoid(all)) st[...]   coord origin, smart route
```

## Caps & strokes

```
line(len(120)) st[($gray.bold,w($stroke.width.soft))]
line(len(120)) cap(n,ar) st[($gray.bold,w($stroke.width.soft))]
line(from(#a), to(#b), intent(flow))                  cap/stroke from intent
```

`cap(start, end)` lands on the first and last nodes. For endpoint form
the start node is `from`, the end node is `to`. For vector form the
start node is `p()`, the end node is at the tip of the rotated segment.

## Rejections

The parser/validator stops these at parse time rather than producing
silent visual bugs at render time:

- `line(100)`: bare numeric. Use `line(len(100))`.
- `line(len(100)) s(W,H)`: `s()` not allowed on line; length lives in `len()`.
- `line(from(...), to(...)) s(W,H)`: same.
- `line(len(100)) rot(45)`: use `angle()` instead.
- `line(from(...), to(...)) rot(45)`: direction comes from the points.
- `line(len(100))` with no stroke and no intent, open paths render nothing.
- `line(len(100), from(#a))`: pick one form, don't mix.
- `connect(#a, #b)`: removed; use `line(from(#a), to(#b))`.

## `line` is a vector

Anything you can do to a vector applies to a line: `f[...]` (no effect on
the open path itself, but used for fills on closed segments if you
construct one), `st[...]`, `cap(...)`, `o(...)`, `flip(...)`. Use `lookup
format:"blueprint"` to inspect, the returned element is a `vector` with
two nodes. The `line(...)` syntax is sugar over the underlying vector;
the underlying vector is the source of truth.
