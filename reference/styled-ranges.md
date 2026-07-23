---
assumes: blueprint/text
dsl: [span, spans, styled-range]
---
# Blueprint Styled Ranges

## Syntax

`spans[...]` continuation line under a text element for mixed formatting:

```
t("Get started for free today",$font.family,$font.size.lg) f[($color.text.primary)] #cta_text
  spans[("Get started",b),("free",b,$color.secondary)]
```

Use `$color.secondary` (the secondary brand callout role) when the highlighted word is a promotional / "look here" emphasis. Reach for palette stops (`$rose.firm`, `$emerald.mid`, `$violet.firm`) when the design specifically wants that hue — see the Key Patterns below for several variations.

**Mods:** `b` bold · `i` italic · `u` underline · `w(N)` weight (`$font.weight.*`) · `s(N)` size (`$font.size.*`) · `f(family)` font · color (token `$ref` or `#hex`) · `o(N)` opacity (`$visibility.*`). In explicit mode `w()`, `s()`, and `o()` are tokenizable slots — use tokens, not bare numerics.

Color slots take tokens just like fills — accent words follow the active brand and mode the same way solid fills do. Reach for palette tokens (`$rose.mid`, `$emerald.mid`, `$violet.firm`) for hue accents and `$neutral.X` / `$slate.X` for typographic emphasis.

**Duplicate words:** Add 0-based occurrence index — `("the",0,b),("the",1,i)`

## Key Patterns

```
t("Build something amazing",$font.family,$font.size.4xl,b) f[($stone.intense)]
  spans[("amazing",s($font.size.5xl),f(Bungee Shade),$orange.mid)]
t("$49/month",$font.family,$font.size.sm) f[($neutral.intense)]
  spans[("$49",s($font.size.3xl),b,$emerald.mid),("/month",$slate.mid)]
t("2,847 users",$font.family,$font.size.2xl,b) f[($neutral.intense)]
  spans[("users",w($font.weight.firm),s($font.size.md),$slate.mid)]
t("The art of modern design",Lora,$font.size.4xl,b) f[($stone.intense)]
  spans[("art",f(Nothing You Could Do),$violet.firm),("modern design",f(Inter),$neutral.intense)]
t("Run npm install to get started",$font.family,$font.size.sm) f[($slate.bold)]
  spans[("npm install",f(Fira Code),$fuchsia.mid)]
t("Free forever · No credit card required",$font.family,$font.size.sm) f[($neutral.mid)]
  spans[("Free forever",b,$slate.intense),("No credit card required",b,$slate.intense)]
```

Always style: hero headlines (accent word/font), prices (large amount + small unit), stats (bold number + lighter label). Usually: feature titles, inline code. Rarely: body. Never: captions.
