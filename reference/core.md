# Blueprint Core

One element per line, 2-space indent = child. Properties are space-separated, in any order.

```
al(v,g($spacing.lg),pad($spacing.xl)) s(360,hug) f[($color.surface)] rd($radius.md) shadow($color.shadow,o($visibility.faint),y(2),blur(8)) "Card"
  t("Settings",$font.family,$font.size.lg,sb) f[($color.text.primary)] #title
  t("Manage your preferences",$font.family,$font.size.sm) s(fill,hug) f[($color.text.secondary)] #desc
  al(h,x(c),y(c),g($spacing.sm),pad($spacing.md,$spacing.lg)) s(fill,hug) f[($color.primary)] rd($radius.sm) "Button"
    svg(icon:check) s(16,16) f[($color.on-primary)]
    t("Save",$font.family,$font.size.sm,sb) f[($color.on-primary)]
```

**Types** (the only valid ones, no `rect`/`div`/`img` aliases): `r` rectangle, `c` circle, `t("text",font,size)`, `line(...)` straight line (sugar over a vector, see `blueprint/lines`), `fr` frame, `gr` group, `al()` auto layout, `svg(icon:name)` Phosphor icon, `v()` vector (charts and freeform paths only), `mask` / `mask(alpha)` / `mask(luminance)` mask frame, `bool(union|subtract|intersect|exclude)` boolean-op frame. Only `fr`/`gr`/`al()`/`mask`/`bool()` take children. A mask frame's TOP child (last in z-order) is the clip silhouette. A plain create or paste into a mask lands BELOW that shape as masked content and never becomes the mask, so appending cannot silently blank the frame. Use `after(#shape)` to make a new element the mask on purpose.

**Props** (exact names, no CSS `width()`/`background()`): `p(x,y)` position (`c` centers, `p(c,c)`, omit on top-level elements so they auto-place beside existing work), `s(w,h)` size with `number`/`fill`/`fill:N`/`hug` (SVG size goes in `s(W,H)`, not `svg()`), `rd(N)` or `rd(TL,TR,BR,BL)` corner radius (add a trailing `smooth(0..1)` inside for iOS-style squircle corners, `rd(12,smooth(0.6))`), `rot(N)`, `o(N)` opacity, `blend(mode)`, `flip(1,0)` (per-axis 1/0 flags horizontal then vertical, both required, letters like `h`/`v` are a silent no-op), `front`/`back`, `pin(H,V)` how a child re-places when its container resizes per axis (H `l|r|c|lr|scale`, V `t|b|c|tb|scale` = left/right/center/stretch/scale edge, default `pin(l,t)`, an empty slot keeps that axis so `pin(,c)` sets only vertical, valid only on a plain-frame child or an `abs` auto-layout child, never a flow child or a group child), and `locked` (no clearer). Five boolean flags each have a `no-` clearer: `clip`/`no-clip`, `isolate`/`no-isolate` (container flattens before blending like Figma's Normal, default is pass-through), `hidden`/`no-hidden` (invisible), `constrain`/`no-constrain` (lock aspect ratio), `abs`/`no-abs` (frees a child from layout flow, position it with `p()`). On a modify line, omitting a flag PRESERVES its current value, so write the `no-` form to clear it.

**Auto layout**: `x`, `y`, `g`, `pad` go inside `al(...)`, comma-separated, `al(v,g($spacing.md),pad($spacing.lg))`. `g()` and `pad()` are always required (`$spacing.none` for zero). Sizing, alignment, and wrapping: see `blueprint/layout`.

**Refs**: omit IDs on new elements. A 16-char hex id or `#ref` as the first token modifies that element, a trailing `#ref` assigns one, and a trailing `"text"` is a NAME (display only, NOT addressable later). Anything you might modify or delete needs a `#ref`, not just a name. `#ref` = hex id everywhere: rows, directives, lookup, export, commands, previewIds, and `<el id="#ref">Name</el>` in replies. Never look one up to get the other, and never write `#` before a hex id. An element you create without a `#ref` gets one from Brilliant (a letter plus a number, `#g1`, `#h12`), returned in the result and usable like any ref you wrote.

**Modify is flat**: one line per element, never indented. A line with no id/ref is always a create. To move an element OR create a child inside an existing one, use `parent(#target)` (see `blueprint/directives`).

**A modify only changes the props you name**, omitted props are kept. On a ROTATED element, `#ref p(x,y)` moves it and `#ref rot(N)` rotates it about its center, both keeping the current size, so you never re-send `s()` alongside `p()`/`rot()`. Send `s()` only to actually resize.

**The canvas tools are the only write path to a canvas**: use `create_modify_elements` / `execute_commands` (or the `<objects>` block in your reply). Never edit a `.bl` or `.design` file with bash, python, or the file tools: the app reloads it from disk, so your write is discarded and every element id you hold goes stale.

**Annotations**: `//` and `--` strip from any line. `// label` also sets an undo checkpoint, and a full-line `// label` is that checkpoint mark on its own line (not a stripped comment). `--` is plain narration. A checkpoint from a block that halts before any line applied does not persist.

**Tokens**: in explicit mode every color, font, and scale slot takes a `$token`, and bare hex or numerics halt the call (see [`design-systems/core`](https://github.com/brilliant-hq/brilliant/blob/main/knowledge/design-systems/core.md)). Text with no `f[]` defaults to `$color.text.primary`, and is `hug` (single line) unless `s(fill,hug)` lets it wrap.
