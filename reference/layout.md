---
assumes: blueprint/core
dsl: [fill, hug]
---
# Blueprint Layout

Every `al()` needs a direction (`al(v)` / `al(h)`) and `g()` + `pad()`
(use `$spacing.none` for zero). Each element is sized per axis: `hug`
fits content, `fill` stretches to the parent, `fill:N` takes a
proportional share, or a fixed number.

```
al(v,g($spacing.lg),pad($spacing.xl)) s(400,hug) "Page"
  al(h,x(sb),y(c),g($spacing.md),pad($spacing.none)) s(fill,hug) "Header"
    t("Logo",$font.family,$font.size.lg,b) f[($color.text.display)]
    al(h,g($spacing.sm),pad($spacing.none)) s(hug,hug) "Nav"
      t("Home",$font.family,$font.size.sm,m) f[($color.text.secondary)]
      t("About",$font.family,$font.size.sm,m) f[($color.text.secondary)]
  al(h,g($spacing.md),pad($spacing.none)) s(fill,hug) "Content"
    al(v,g($spacing.md),pad($spacing.none)) s(fill:2,hug) "Main"
      t("Welcome back",$font.family,$font.size.xl,sb) f[($color.text.primary)]
      t("Here is the news",$font.family,$font.size.sm) s(fill,hug) f[($color.text.secondary)]
    al(v,g($spacing.md),pad($spacing.md)) s(fill:1,hug) f[($color.surface.container)] rd($radius.sm) "Sidebar"
      t("Quick stats",$font.family,$font.size.sm,sb) f[($color.text.primary)]
```

`x()` aligns on the horizontal axis, `y()` on the vertical; values are
`s`/`c`/`e`/`sb` and stay physical (never flipped by direction). `sb`
(space-between) is main-axis only. `fill:2` beside `fill:1` splits 2/3
and 1/3.

**Sizing** asks "who decides this size?": content picks `hug` (text,
buttons), the parent picks `fill`, a spec picks a fixed number. A `fill`
child inside a `hug` ancestor on the same axis falls back to its content
size. Prose text needs `s(fill,hug)` so it wraps rather than overflows.

**Min/max**: `min(w,h)` and `max(w,h)` clamp an element's size. They
bound a `hug` result and floor/cap a `fill` child, with any excess
redistributed to siblings; `min` wins over `max` when they conflict. An
empty slot leaves that axis unconstrained: `min(200,)` sets width only,
`min(,120)` height only, and `min(200)` is width only. Bare `min()` /
`max()` clear that kind. Write them right after `s()`, e.g.
`s(fill,hug) min(200,) max(600,400)`.

**Wrap**: `al(h,wrap,...)` flows overflowing children onto the next row;
it needs a fixed or fill main-axis size. `g(main,cross)` sets the gap
between wrap rows separately (`g($spacing.sm,$spacing.xl)`); `g(N)` uses
N for both.

**Centering and `abs`**: `p(c,c)` centers a child, `p(c,20)` / `p(20,c)`
center one axis. Inside an `al()` a child must be `abs` for `p()` to
apply; `abs` lifts it out of layout flow entirely (badges, dots,
floating overlays) without affecting siblings or hug sizing. Inside a
`fr` or `gr`, `p()` already positions children freely.

**Pin** (`pin(h,v)`): how a child holds its place when the container is
resized. Per axis five kinds: `l`/`t` hold the leading edge (min, the
default), `r`/`b` hold the trailing edge (max), `c` holds the center,
`lr`/`tb` hold both edges so the child stretches, and `scale`
interpolates both edges proportionally. Absent is `pin(l,t)` (today's
anchor-to-top-left), so it never serializes; an empty slot skips that
axis (`pin(,c)` sets only the vertical). Write it beside `p()`. Pins
apply to children of a `fr`, component, instance, or non-hug `gr`; a hug
`gr` has none until you resize it (which fixes its size). Inside an
`al()`, only an `abs` child takes a pin, and it pins within the frame:

```
r p(260,12) s(20,20) pin(r,t) "close button"
r p(12,90) s(276,14) pin(lr,b) "bottom bar"
al(v,g(8),pad(12)) abs p(16,16) pin(r,b) "floating badge"
```
