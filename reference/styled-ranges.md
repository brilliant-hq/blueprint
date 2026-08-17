---
assumes: blueprint/text
dsl: [span, spans, styled-range, link]
---
# Blueprint Styled Ranges

## Syntax

`spans[...]` continuation line under a text element for mixed formatting:

```
t("Get started for free today",$font.family,$font.size.lg) f[($color.text.primary)] #cta_text
  spans[("Get started",b),("free",b,$color.secondary)]
```

Use `$color.secondary` (the secondary brand callout role) when the highlighted word is a promotional / "look here" emphasis. Reach for palette stops (`$rose.firm`, `$emerald.mid`, `$violet.firm`) when the design specifically wants that hue, see the Key Patterns below for several variations.

**Mods:** `b` bold · `i` italic · `u` underline · `strike` strikethrough · `w(N)` weight (`$font.weight.*`) · `s(N)` size (`$font.size.*`) · `f(family)` font · color (token `$ref`, `#hex`, or a full gradient spec, see Range paint below) · `o(N)` opacity (`$visibility.*`) · `ls(N)` letter spacing (bare pixels) · `link("url")` hyperlink. In explicit mode `w()`, `s()`, and `o()` are tokenizable slots, use tokens, not bare numerics.

Color slots take tokens just like fills, accent words follow the active brand and mode the same way solid fills do. Reach for palette tokens (`$rose.mid`, `$emerald.mid`, `$violet.firm`) for hue accents and `$neutral.X` / `$slate.X` for typographic emphasis.

**Duplicate words:** Add 0-based occurrence index, `("the",0,b),("the",1,i)`

## Range paint: gradients and links

The span color slot takes the same nested paint spec a fill or stroke does, so a range can carry a gradient, not just a solid: `linear(...)`, `radial(...)`, `angular(...)`, `diamond(...)` all work in place of a `$ref`/`#hex` color (full syntax in `blueprint/paint`). A range gradient is framed by the WHOLE text box, so the range shows the slice of one continuous gradient spanning the text (the same handles a fill gradient uses).

```
t("50% OFF EVERYTHING",$font.family,$font.size.4xl,b) f[($neutral.intense)]
  spans[("50% OFF",linear(90,$rose.mid,$amber.mid))]
```

`link("url")` turns a range into a hyperlink. The URL is always quoted (`link("https://brilliant.design")`). A link is pure data: it does NOT restyle the glyphs, so add `u` and a color yourself if you want the classic underlined-link look.

```
t("Read the docs to get started",$font.family,$font.size.sm) f[($color.text.secondary)]
  spans[("docs",u,link("https://brilliant.design/docs"))]
```

Only solids and gradients are available on a substring; shader and image fills stay whole-element only.

**Where each renders.** Gradient ranges and links both render on the canvas. On export a link becomes a real anchor in HTML (`<a href>`) and SVG (`<a xlink:href>`), while PDF and the raster formats (PNG/JPEG/WebP) cannot carry a link, so linked text exports as ordinary styled text. A gradient range renders in the raster formats (which read back the live canvas) but the SVG, PDF, and HTML lanes fall back to that range's base text color.

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
