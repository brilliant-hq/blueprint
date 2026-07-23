---
assumes: blueprint/paint
dsl: [shadow, outerglow, eblur, inner, glass]
---
# Blueprint Effects

Two systems, don't mix them up.

## Element-level (standalone properties)
```
shadow($neutral.intense,o($visibility.subtle),y(2),blur(8))                          ← drop shadow
shadow($neutral.intense,o($visibility.faint),blur(4)) shadow($neutral.intense,o($visibility.subtle),y(8),blur(24))  ← layered
outerglow($emerald.mid,o($visibility.soft),blur(16))                           ← outer glow
eblur(4)                                                                   ← element blur
```

Params (all optional): color (token `$ref` or hex) · `o(N)` opacity (token or 0–1) · `x(N)` · `y(N)` offsets · `blur(N)` · `sp(N)` spread · `blend(mode)`. Stack multiple on one element.
Defaults, `shadow()`: `$color.shadow o(0.25) y(4) blur(8)`. `outerglow()`: `$color.glow o($visibility.firm) blur(8)`. Color slots default to semantic aliases (`$color.shadow` → `$neutral.intense`, `$color.glow` → `$neutral.hint` in the catalog), so bare effects pick up brand changes automatically.

**Color and opacity slots both accept tokens.** A token-bound shadow follows brand / mode switches when you bind it to a semantic alias: `shadow($color.shadow, ...)` re-resolves through `theme.dark` if the active mode flips, picking up whatever stop the catalog maps the alias to per mode. Reach for `$visibility.{hint, faint, subtle, soft, mid, firm, bold, strong, intense}` (catalog-bound 9-stop scale from 0.02 to 1.0) instead of arbitrary decimals.

Complete card with layered elevation:
```
al(v,g($spacing.md),pad($spacing.md)) s(300,hug) f[($color.surface)] rd($radius.md) shadow($neutral.intense,o($visibility.faint),y(1),blur(2)) shadow($neutral.intense,o($visibility.subtle),y(8),blur(24)) #elev_card
  t("Elevation",$font.family,$font.size.md,sb) f[($color.text.primary)] #elev_title
  t("Two-layer shadow creates realistic depth",$font.family,$font.size.sm) s(fill,hug) f[($color.text.secondary)] #elev_desc
```

## Fill-type (inside `f[...]` array)
```
f[($color.surface.container),(f2,inner($neutral.intense,o($visibility.faint),y(1),blur(2)))]    ← inner shadow
f[($color.surface),(f2,glow($color.secondary,o($visibility.firm),blur(8)))]                  ← inner glow
f[(solid($color.on-surface,o($visibility.subtle))),(f2,blur(12))]                            ← background blur
f[(glass)] · f[(glass(chroma(0.5),frost(8)))]                                                ← liquid glass (refracts backdrop; see `blueprint/paint`)
```

Fill-type effects stack with other fills in z-order. If you're tempted to create a separate element for a visual effect, it's almost certainly a fill layer instead.

The color and opacity slots inside `inner()`, `glow()` and the wrapping `solid()` all accept tokens too, same rules as drop shadow and outer glow.
