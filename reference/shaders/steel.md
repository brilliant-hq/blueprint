---
assumes: blueprint/shaders/overview
dsl: [steel, mercury, distortion, depth]
---
# Shader: Liquid Steel

`steel()` is flowing chrome or liquid mercury: clean industrial
character, less chromatic than `metal()`, more fluid.

## Parameters

`flow(N)` surface fluidity, `roughness(N)` grain (higher is more matte),
`distortion(N)` warp intensity, `depth(N)` reflection intensity,
`angle(N)` light direction, `speed(N)` animation. Colors are positional
token args; default `steel()` is chrome.

## Colors

Shader colors are tokens. Chrome and steel tones come from Tailwind
neutrals (`$zinc.*`, `$slate.*`, `$stone.*`). For a tuned recipe (rose
gold, blue steel) that no Tailwind stop matches, extend the active DS
with `art.*` primitive tokens and reference them as `$art.<name>`.

## Example

Steel is at its best on a stroke. A steel-edged button:

```
al(h,x(c),y(c),g($spacing.none),pad($spacing.md,$spacing.lg)) s(hug,hug) f[($color.surface)] st[(steel($slate.strong,$slate.subtle,flow(0.6),distortion(0.9),speed(2)),w($stroke.width.mid))] rd($radius.full) "Subscribe"
  t("Subscribe",$font.family,$font.size.sm,sb) f[($color.text.primary)]
```

It also works as a low-opacity texture fill (`steel(...,opacity(0.2))`)
layered under a solid surface fill.

## Best uses

Like `metal()`, use steel on strokes and decorative elements, never
under text. Its flowing quality suits borders and accent bars.
