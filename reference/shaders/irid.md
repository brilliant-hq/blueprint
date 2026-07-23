---
assumes: blueprint/shaders/overview
dsl: [irid, spread, intensity, metallic]
---
# Shader: Iridescent

Iridescent liquid crystal — deep shifting colors with metallic surface tension. The default (Oil Slick) produces a dark, luxurious look. Supports 3 colors.

## Parameters

`intensity(N)` color strength · `spread(N)` fold density · `metallic(N)` metallic vs matte · `noise(N)` surface complexity · `speed(N)` shimmer rate · Custom colors as positional args

## Recommended Palettes

Tailwind stops cover most recipes directly. Tuned ones get an `$art.*` extension on the active DS.

- `irid()` — Oil Slick (default: deep purple/teal/violet)
- `irid($indigo.intense,$orange.mid,$violet.mid)` — Deep Sapphire
- `irid($zinc.intense,$emerald.mid,$cyan.mid)` — Dark Emerald
- `irid($slate.intense,$indigo.mid,$purple.mid)` — Void Indigo
- `irid($pink.mid,$purple.mid,$indigo.mid)` — Pink Nebula
- `irid($emerald.mid,$cyan.mid,$amber.mid)` — Aqua Prism
- Tuned recipe: extend DS with `art.pearl.{1..3}` and reference

## Examples

**Iridescent badges (shader on stroke, text stays readable):**
```
al(h,x(c),y(c),g($spacing.sm),pad($spacing.sm,$spacing.md)) s(hug,hug) f[($slate.intense)] st[(irid($pink.mid,$purple.mid,$indigo.mid),w($stroke.width.mid))] rd($radius.sm) shadow($purple.mid,o($visibility.soft),blur(16)) #badge_premium
  svg(icon:star) s(12,12) f[($indigo.faint)] #badge_icon
  t("Premium",$font.family,$font.size.xs,sb) f[($indigo.faint)] #badge_label
al(h,x(c),y(c),g($spacing.sm),pad($spacing.sm,$spacing.md)) s(hug,hug) f[($stone.intense)] st[(irid($amber.mid,$red.mid,$pink.mid),w($stroke.width.mid))] rd($radius.full) shadow($red.mid,o($visibility.soft),blur(12)) #badge_hot
  svg(icon:fire) s(12,12) f[($amber.subtle)] #hot_icon
  t("Hot",$font.family,$font.size.xs,sb) f[($amber.subtle)] #hot_label
al(h,x(c),y(c),g($spacing.none),pad($spacing.sm,$spacing.md)) s(hug,hug) f[(irid($emerald.mid,$cyan.mid,$amber.soft,shape(metaballs),intensity(0.90),spread(0.45),noise(0.50),metallic(0.40),speed(0.70),opacity(0.30)))] st[(solid($neutral.hint,o($visibility.soft)),w($stroke.width.subtle))] rd($radius.full) #badge_tropical
  t("Tropical",$font.family,$font.size.xs,sb) f[($neutral.hint)] #tropical_label
```

**Glass card over iridescent background** (real refractive `glass` fill — see `effects/glass`):
```
fr s(360,220) f[($neutral.intense),(f2,irid($pink.mid,$purple.mid,$indigo.mid))] rd($radius.lg) clip "Hero" #hero
  al(v,y(c),x(c),g($spacing.md),pad($spacing.lg)) s(280,hug) f[(glass(frost(4),chroma(0.4)))] st[(solid($neutral.hint,o($visibility.soft)),w($stroke.width.subtle))] rd($radius.lg) shadow($neutral.intense,o($visibility.soft),y(12),blur(32)) abs p(40,c) "Glass" #glass
    t("Pink Nebula",$font.family,$font.size.xl,sb,align(c)) f[($neutral.hint)] #title
    t("Custom palette",$font.family,$font.size.sm,align(c)) f[(solid($neutral.hint,o($visibility.firm)))] #sub
```

## Best Uses

Dark palettes produce the most striking results. Use on strokes `st[(irid(),w($stroke.width.mid))]` for buttons, as a background with dim overlay, or as low-opacity texture under a solid fill `f[($stone.intense),(f2,irid(opacity(0.15)))]`.
