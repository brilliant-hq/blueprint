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

**Mods:** `b` bold · `i` italic · `u` underline · `strike` strikethrough · `w(name)` weight by name (`w(semibold)`, `w(medium)`; numeric weights like `w(600)` work only in `none` mode, and `$font.weight.*` tokens are not supported on a range) · `s(N)` size (bare pixels in every mode; `$font.size.*` tokens are not supported on a range) · `f(family)` font (literal name or `$font.family.*` token) · color (token `$ref`, `#hex`, or a full gradient spec, see Range paint below) · `o(N)` opacity (`$visibility.*` token; bare numerics only in `none` mode) · `ls(N)` letter spacing (bare pixels, `none` mode only) · `link("url")` hyperlink. In explicit mode the tokenizable span slots are color, `f()`, and `o()`; sizes stay bare pixels, weights stay names, and `ls()` is refused (letter spacing currently has no token form; use `none` mode when a design needs it).

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

**Where each renders.** Gradient ranges and links both render on the canvas. A gradient range now exports at full fidelity in every lane: the raster formats (PNG/JPEG/WebP) read back the live canvas, SVG and PDF outline just the gradient glyphs to vector paths filled with the gradient (Figma's move) while the rest of the text stays real text, and HTML/React paint the span with `background-clip:text`. On export a link becomes a real anchor in HTML (`<a href>`) and SVG (`<a xlink:href>`), while PDF and the raster formats do not carry a link (the PDF exporter draws glyphs, not link annotations), so linked text exports as ordinary styled text.

## Key Patterns

```
t("Build something amazing",$font.family,$font.size.4xl,b) f[($stone.intense)]
  spans[("amazing",s(48),f(Bungee Shade),$orange.mid)]
t("$49/month",$font.family,$font.size.sm) f[($neutral.intense)]
  spans[("$49",s(36),b,$emerald.mid),("/month",$slate.mid)]
t("2,847 users",$font.family,$font.size.2xl,b) f[($neutral.intense)]
  spans[("users",w(semibold),s(16),$slate.mid)]
t("The art of modern design",Lora,$font.size.4xl,b) f[($stone.intense)]
  spans[("art",f(Nothing You Could Do),$violet.firm),("modern design",f(Inter),$neutral.intense)]
t("Run npm install to get started",$font.family,$font.size.sm) f[($slate.bold)]
  spans[("npm install",f(Fira Code),$fuchsia.mid)]
t("Free forever · No credit card required",$font.family,$font.size.sm) f[($neutral.mid)]
  spans[("Free forever",b,$slate.intense),("No credit card required",b,$slate.intense)]
```

Always style: hero headlines (accent word/font), prices (large amount + small unit), stats (bold number + lighter label). Usually: feature titles, inline code. Rarely: body. Never: captions.
