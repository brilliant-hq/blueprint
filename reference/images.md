---
assumes: blueprint/paint
dsl: [img, icon:, recolor_children]
---
# Blueprint Images & SVGs

## Image fills

`f[(img(URL or assetPath))]` places an image as a fill; put it on a rect
and `clip`: `r s(400,300) f[(img(...))] rd($radius.md) clip "Hero"`.
Handy sources: `picsum.photos/id/{n}/800/400` (photos),
`i.pravatar.cc/150?img={n}` (avatars),
`ui-avatars.com/api/?name=John+Smith` (letter avatars). `generate_image`
returns an `assetPath` you feed straight into `img()`.

Optional args after the source, any order: `img(src, mode, o(N))`. `mode`
is `fill` (default, covers the element, excess clipped), `fit` (letterboxed
inside the element, margins transparent), `crop` (free positioning), or
`repeat` (tiles at natural pixel size from the top-left). `o(N)` sets image
opacity (0-1). `crop` mode also takes `crop(x0,y0,x1,y1,x2,y2,x3,y3)` — the
four crop-window corners, as emitted by `lookup`. `repeat` mode takes
`scale(N)` to size the tile (e.g. `img(tile.png, repeat, scale(0.16))`).
All render on canvas and in exports. Scale modes and interactive crop are
detailed in `reference/crop`.

## SVG import

`svg(icon:house)` for a bundled Phosphor icon, `svg(https://...)` for a
URL (fetched at import), `svg(/path)` for a local file, `svg(<svg>...)`
for inline markup (collapsed to one line). A fill or stroke on the `svg`
line overrides the imported paint and accepts every fill type
(`svg(icon:house) f[($orange.mid)]`, `svg(...) f[(metal())]`); recolor
after creation with the `recolor_children` command. Size goes in `s()`.

**Two Phosphor weights bundled:** regular (`svg(icon:play)`) and fill
(`svg(icon:play-fill)`). Reach for fill on active/selected states, brand
glyphs, or any place a solid shape reads better than an outline; stick
with regular for neutral UI affordances. Only those two weights ship;
`-bold`/`-light`/`-thin`/`-duotone` silently fall back to regular, so
don't write them. Other libraries load by URL: Lucide
(`unpkg.com/lucide-static/icons/{name}.svg`), Simple Icons for brands,
Wikimedia for logos and flags, or DiceBear for avatars
(`api.dicebear.com/9.x/notionists/svg?seed={name}`).

## Rules

Every icon is an SVG, never a `vector` element and never an emoji. Use
`generate_image` for photos, product shots, and textures (see
`images/prompts`).
