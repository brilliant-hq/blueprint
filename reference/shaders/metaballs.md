---
assumes: blueprint/shaders/overview
dsl: [metaballs, count, density, speed]
---
# Shader: Metaballs

Organic fluid simulation: blobs merge and separate with surface tension,
like a lava lamp or oil in water.

## Parameters

`count(N)` blob count (fewer is cleaner), `size(N)` blob scale (larger is
calmer), `speed(N)` energy. Colors are positional token args. Size and
count are coupled, so pick a density:

- **Detailed**: `size(0.2),count(15),speed(0.4)`, complex and textured.
- **Balanced**: `size(0.3),count(10),speed(1)`, the default.
- **Organic**: `size(0.7),count(5),speed(0.35)`, calm and natural.
- **Ambient**: `size(0.85),count(4),speed(0.3)`, meditative background.

## Colors

Shader colors are tokens. Tailwind stops cover most recipes:
`metaballs($zinc.intense,$orange.mid,$violet.mid,$cyan.mid)` is vibrant,
`metaballs($amber.mid,$red.mid,$pink.mid)` a warm sunset,
`metaballs($amber.hint,$amber.faint,$amber.soft,$amber.mid)` honey gold.
For a tuned recipe, extend the DS with `art.*` tokens.

## Examples

As a section background the metaballs fill goes straight on the frame,
no child rect (see `blueprint/shaders/overview` for placing content over
a shader). As a small decorative accent:

```
r s(4,48) f[(metaballs($amber.mid,$red.mid,$pink.mid))] rd($radius.xs) "Bar"
```
