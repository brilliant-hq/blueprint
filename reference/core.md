# Blueprint Core

One element per line; 2-space indent = child. Properties are
space-separated, in any order.

```
al(v,g($spacing.lg),pad($spacing.xl)) s(360,hug) f[($color.surface)] rd($radius.md) shadow($color.shadow,o($visibility.faint),y(2),blur(8)) "Card"
  t("Settings",$font.family,$font.size.lg,sb) f[($color.text.primary)] #title
  t("Manage your preferences",$font.family,$font.size.sm) s(fill,hug) f[($color.text.secondary)] #desc
  al(h,x(c),y(c),g($spacing.sm),pad($spacing.md,$spacing.lg)) s(fill,hug) f[($color.primary)] rd($radius.sm) "Button"
    svg(icon:check) s(16,16) f[($color.on-primary)]
    t("Save",$font.family,$font.size.sm,sb) f[($color.on-primary)]
```

**Types** (the only valid ones; no `rect`/`div`/`img` aliases): `r`
rectangle, `c` circle, `t("text",font,size)`, `line(...)` straight line
(sugar over a vector, see `blueprint/lines`), `fr` frame, `gr` group,
`al()` auto layout, `svg(icon:name)` Phosphor icon, `v()` vector
(charts and freeform paths only), `mask` / `mask(alpha)` /
`mask(luminance)` mask frame (its TOP child, last in z-order, is the clip
silhouette; a plain create/paste into a mask lands BELOW the shape as masked
content and never becomes the mask, so appending can't silently blank the
frame; `after(#shape)` to make a new element the mask on purpose),
`bool(union|subtract|intersect|exclude)` boolean-op frame. Only
`fr`/`gr`/`al()`/`mask`/`bool()` take children.

**Props** (exact names; no CSS `width()`/`background()`): `p(x,y)`,
`s(w,h)` with `number`/`fill`/`fill:N`/`hug`, `rd(N)` or
`rd(TL,TR,BR,BL)` (add a trailing `smooth(0..1)` inside for iOS-style
squircle corners: `rd(12,smooth(0.6))`), `rot(N)`, `o(N)`, `clip`,
`isolate` (container flattens before blending, like Figma's Normal;
default is pass-through; `no-isolate` clears it), `blend(mode)`,
`flip(1,0)` (per-axis 1/0 flags: horizontal, vertical; both required, letters like `h`/`v` are a silent no-op), `front`/`back`,
`abs` (frees a child from layout flow, position it with `p()`),
`pin(H,V)` (how a child re-places when its container is resized, per axis:
H `l|r|c|lr|scale`, V `t|b|c|tb|scale` = left/right/center/stretch/scale
edge; default `pin(l,t)`; a slot left empty keeps that axis: `pin(,c)` sets
only vertical; only valid on a plain-frame child, or an `abs` auto-layout
child, never a flow child or a group child),
`hidden` (invisible; `no-hidden` clears), `locked`, `constrain` (lock
aspect ratio; `no-constrain` clears). `c` in
`p()` centers: `p(c,c)`. Omit `p()` on top-level elements; they
auto-place beside existing work. SVG size goes in `s(W,H)`, not `svg()`.

**Auto layout**: `x`, `y`, `g`, `pad` go inside `al(...)`, comma-
separated: `al(v,g($spacing.md),pad($spacing.lg))`. `g()` and `pad()`
are always required (`$spacing.none` for zero). Sizing, alignment, and
wrapping: see `blueprint/layout`.

**Refs**: omit IDs on new elements. A 16-char hex id or `#ref` as the
first token modifies that element; a trailing `#ref` assigns one. A
trailing `"text"` is a NAME, display only, NOT addressable later.
Anything you might modify or delete needs `#ref`, not just a name.
`#ref` = hex id everywhere: rows, directives, lookup, export, commands,
previewIds, and `<el id="#ref">Name</el>` in replies. Never look one up
to get the other; never write `#` before a hex id.

**Modify is flat**: one line per element, never indented. A line with no
id/ref is always a create. To move an element OR create a child inside
an existing one, use `parent(#target)` (see `blueprint/directives`). An
indented leading-`#ref` line whose element is NOT already a child of the
line above is refused, never silently moved -- use `parent(#target)`.

**The canvas tools are the only write path to a canvas**: use
`create_modify_elements` / `execute_commands` (or the `<objects>` block in your
reply). Never edit a `.bl` or `.design` file with bash, python, or the file
tools: the app reloads it from disk, so your write is discarded and every
element id you hold goes stale.

**A modify only changes the props you name**; omitted props are kept. On a
ROTATED element, `#ref p(x,y)` moves it and `#ref rot(N)` rotates it about
its center, both keeping the current size, so you never need to re-send `s()`
alongside `p()`/`rot()` to hold the size. Send `s()` only to actually resize.

**Annotations**: `//` and `--` strip from any line. `// label` also sets
an undo checkpoint; `--` is plain narration. A full-line `// label` is that
checkpoint mark on its own line, not a stripped comment. A checkpoint from a
block that halts before any line applied does not persist.

**Tokens**: in explicit mode every color, font, and scale slot takes a
`$token`; bare hex or numerics halt the call (see [`design-systems/core`](https://github.com/brilliant-hq/brilliant/blob/main/knowledge/design-systems/core.md)).
Text with no `f[]` defaults to `$color.text.primary`, and is `hug`
(single line) unless `s(fill,hug)` lets it wrap.
