---
assumes: blueprint/core
dsl: [t(, lh, ls, ps, pi, eb, bl, valign, strike, case(, clamp(, feat(]
---
# Blueprint Text

```
t("Dashboard",$font.family,$font.size.xl,b) f[($color.text.primary)]
t("Chapter One",$font.family.serif,$font.size.lg,italic) f[($violet.mid)]
t("LAUNCH DAY",$font.family,$font.size.xs,sb,ls($font.letterSpacing.loose)) f[($color.error)]
t("Updated 5 min ago",$font.family,$font.size.xs,m,align(r)) s(fill,hug) f[($color.text.disabled)]
t("Long description that wraps",$font.family,$font.size.sm,lh($font.lineHeight.snug)) s(fill,hug) f[($color.text.secondary)]
t("Hello\nWorld",Fira Code,$font.size.md) f[($emerald.mid)]
t("SUMMER SALE",$font.family,$font.size.lg,b,case(upper),strike) f[($color.error)]
t("Terms and conditions apply to this offer",$font.family,$font.size.sm,clamp(2)) s(fill,hug) f[($color.text.secondary)]
t("Outlined",$font.family,$font.size.2xl,b) f[(solid($color.surface,o($visibility.invisible)))] st[($color.text.primary,w($stroke.width.mid),pos(o))]
t("Glow",$font.family,$font.size.2xl,b) f[($color.on-primary)] shadow($color.secondary,blur(12))
t("Blue Title",$font.family,$font.size.4xl,b) f[($color.text.display)]
t("Pink Callout",$font.family,$font.size.4xl,b) f[($color.text.display.alt)]
```

First 3 positional: `t("content",family,size)`, then any order:
Weights: `r`(400) body/descriptions · `m`(500) labels/captions · `sb`(600) subheadings/nav/card titles · `b`(700) headings/hero/CTAs · `eb`(800) display headlines · `bl`(900) watermarks, weight keywords, or `$font.weight.{hint..intense}` tokens. Align: `align(l)` left · `align(c)` center · `align(r)` right · `align(j)` justify. Vertical: `valign(t|c|b)` places the text block top/center/bottom inside a fixed-height box (default top; hug height makes it moot). Direction: `rtl` or `ltr` (default), RTL auto-defaults to right alignment unless overridden. `lh(N)` line-height multiplier, token-bound, use `$font.lineHeight.{none,tight,snug,normal,relaxed,loose}` (e.g. `lh($font.lineHeight.snug)`). Bare `lh(N)` is a 1.0–3.0 multiplier, not pixels; a value > 3 is read as px and divided by the font size. `lh(auto,X)` means the line height is automatic — it follows the font, and `X` is the multiplier the text renders with everywhere; saved documents carry this form for text whose line height was never set. Keep it as-is unless you want a specific value: writing `lh(N)` makes the line height explicit and stops it following the font. `ls(N)` letter spacing, token-bound, use `$font.letterSpacing.{none,tight,snug,normal,relaxed,loose}` (e.g. `ls($font.letterSpacing.loose)`); the scale carries both tight (negative) and loose (positive) tracking. A bare `ls(N)` is pixels; `ls(Nem)` for em. `ps(N)` paragraph spacing, pixels of vertical space between paragraphs (hard `\n` breaks); no space after the last. `pi(N)` paragraph indent, pixels the first line of each paragraph shifts along its leading edge. Both plain pixels, default 0. `li(...)` list markers, one token per paragraph: `u` bullet, `o` numbered, `-` none, with an optional nesting level digit (`u2`, `o3`). A single token applies to every paragraph: `li(u)` bullets the whole text; `li(u,u,-,o)` maps paragraph by paragraph. Numbered markers cycle 1. then a. then i. by level. `lsp(N)` list spacing in pixels: the gap used instead of `ps(N)` between two consecutive list items. `italic`, `underline`, `strike` (strikethrough). `case(upper|lower|title|none)` transforms the displayed casing without changing stored content. `clamp(N)` truncates to at most `N` lines with an ellipsis (`N` ≥ 1); pair it with `s(fill,hug)` so the text wraps first. `feat(...)` sets OpenType font features as a comma list of `tag` or `tag:value` pairs, e.g. `feat(smcp,ss01:1)` (small caps plus stylistic set 1). Escapes: `\"` quote, `\uXXXX` unicode.

Hundreds of curated Google Fonts are available; only Inter ships bundled, the rest download on first use. Unavailable fonts silently fall back. Omit `s()` for short labels (hug). Use `s(fill,hug)` for any text that could exceed its parent's width.

**Modify only some properties**: leave a positional empty to keep its current value:
```
abc123 t(,,,lh($font.lineHeight.normal))   # change only line-height
abc123 t(,Inter,)             # change only font family
abc123 t(,,$font.size.md,b)   # change only size + weight
abc123 t("New title")         # change only content
```
Empty positional (no characters) = preserve. **Whitespace IS literal content**, `t( ,Inter)` sets text to a single space, `t(" ")` does the same. Quoted empty (`t("")`) is rejected, text content cannot be empty.

⚠ Text fill defaults to `$color.text.primary` (mode-aware: `$neutral.intense` in light, `$neutral.hint` in dark), readable across modes even when you forget `f[(...)]`. Still always set the fill explicitly when you want a specific color.
⚠ **Text overflows when hug width exceeds parent width.** Text defaults to `hug` (single line), correct for labels, values, headings. When text content is wider than its parent's resolved width, it overflows. Use `s(fill,hug)` on descriptions, subtitles, and paragraphs so they wrap instead of overflowing.
⚠ **Empty text is invalid.** The app auto-removes empty text elements; the parser rejects creates without content and modifies that explicitly empty the text. To clear a text element, delete it (`delete(#ref)`).
⚠ **`lh()` / `ls()` / `ps()` / `pi()` / `li()` / `lsp()` / `align()` only live inside `t()` props.** Writing them as siblings after the closing paren halts the line. `t("Hi",Inter,16,b,lh($font.lineHeight.tight))` ✓, never `t(...) lh(...)`.
