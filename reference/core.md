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
`mask(luminance)` mask frame (last child is the mask shape),
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
`hidden` (invisible), `locked`, `constrain` (lock aspect ratio;
`no-constrain` clears). `c` in
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

**Modify is flat**: one line per element, never indented. A line with no
id/ref is always a create. To move an element OR create a child inside
an existing one, use `parent(#target)` (see `blueprint/directives`).

**Annotations**: `//` and `--` strip from any line. `// label` also sets
an undo checkpoint; `--` is plain narration.

**Tokens**: in explicit mode every color, font, and scale slot takes a
`$token`; bare hex or numerics halt the call (see [`design-systems/core`](https://github.com/brilliant-hq/brilliant/blob/main/knowledge/design-systems/core.md)).
Text with no `f[]` defaults to `$color.text.primary`, and is `hug`
(single line) unless `s(fill,hug)` lets it wrap.
