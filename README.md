# Blueprint: Language Specification

Blueprint is Brilliant's design language: a concise, lossless, versioned text representation of a
design. It is the canonical storage format for Brilliant designs and the surface both humans and
agents read, write, review, and diff. This document is the authoritative language reference. The
detailed authoring guides live alongside it (`knowledge/blueprint/*.md`), and `objects-syntax.md`
covers how agents emit Blueprint in a chat reply.

## 1. Design goals

- **Lossless.** The same Blueprint always resolves to the same design, byte-exact, forever. Blueprint
  is a unique identifier of a design, and every property of every element round-trips.
- **Minimal.** The least data that losslessly represents a design. Derived state (bounding boxes,
  resolved layout positions) is recomputed rather than stored, and defaults are elided.
- **Concise and readable.** One element per line, terse operators, legible to humans and agents alike.
- **Versioned.** A stored document carries a grammar version, and the language evolves under an
  explicit backwards-compatibility regime (see §7).

## 2. Lexical structure: the four operators

The entire grammar is built on four token shapes. Nothing else is punctuation:

| Shape | Meaning |
|---|---|
| `( )` | grouping, arguments, and tagged parameters `name(value)`: `p(x,y)`, `w(2)`, `rd(8)`, `pos(o)` |
| `[ ]` | lists and stacks: `f[...]` (fills), `st[...]` (strokes), `nodes[...]`, `edges[...]`, `axes[...]`, `spans[...]` |
| `- + ->` | edit operators inside stacks (remove, append, swap), and directed edge walks in vector regions (`0+1-2+`) |

Plus three cross-cutting idioms: `$token` (design-system binding), `#ref` (element identity), and `:`
tagged literals that make a token self-contained (`Inter:$font.family`, `o(0.42:$opacity.40)`).

Structure: one element per line, two-space indent means child. Properties are space-separated in any
order after the type token. `//` and `--` are line annotations (`// label` also marks an undo
checkpoint).

## 3. Element types

A closed set (no CSS-style aliases): `r` rectangle, `c` circle, `t("text",font,size)`, `line(...)`,
`fr` frame, `gr` group, `al(...)` auto-layout, `svg(icon:name)` icon, `v(...)` vector. Only `fr`, `gr`,
and `al()` take children.

## 4. Properties (summary; see the authoring guides for the full surface)

- **Geometry**: `p(x,y)`, `s(w,h)` (`number`/`fill`/`fill:N`/`hug`), `rot(N)`, `o(N)` opacity,
  `flip(h,v)`, `rd(...)` corner radii, `min()`/`max()`, `abs`, `clip`, `isolate`, `blend(mode)`.
- **Paint**: `f[...]` compositing fill stack (solid, gradients, shaders, image, glass, effect fills),
  `st[...]` stroke stack (`pos(c|i|o)`, per-side `w(t,r,b,l)`, caps, dashes). See `paint.md`.
- **Layout**: `al(dir, g(), pad(), x(), y(), wrap)`, per-child sizing/flex/min-max. See `layout.md`.
- **Text**: `t(...)` with weights, alignment, line/letter spacing, paragraph controls, per-run `spans[]`.
  See `text.md`.
- **Effects**: element-level `shadow`/`outerglow`/`eblur` and fill-type `inner`/`glow`/`blur`/`glass`.
- **Vectors**: `v(nodes[(id,x,y,type)], edges[(id,a,b,handles)], regions[...])`, the planar-graph form
  (see §6). **Components**: `comp`/`axes[]`/`variant()`/`inst()`/`at()`/`override()`/`slot`.
- **Design-system binding**: every color, font, and scale slot takes a `$token`.

## 5. Semantics: forgiving authoring, strict canonical

Blueprint has one grammar and two dialects:

- **Authoring** (what agents and humans write) is forgiving. A preprocessor absorbs common-intent
  variants (CSS colors, `px` units, near-miss forms), applies them, and emits a one-line diagnostic
  teaching the canonical form. Two guards keep this honest: a form is absorbed only if it was
  previously an error (so it cannot collide with a valid program), and never if a capable author
  might mean it intentionally.
- **Canonical** (what is stored) is strict. There is no normalization on load, every value is
  self-contained (tokens carry their literal), and element identities are preserved. This is what
  makes a stored document a stable unique identifier. Terseness is preserved in both dialects; only
  the load behavior differs.

The build pipeline is parse, then validate, then build. Derived state (auto-layout positions, hug/fill
extents, component instance subtrees) is reconstructed by the builder, never stored.

## 6. Vectors are planar graphs

A vector is not a path string. It is a graph: nodes (shared, arbitrary valence), edges (straight or
bezier, with per-endpoint handles and per-node type `st`/`mi`/`as`/`di`), and regions (closed faces
that take fills, with explicit hole flags). The canonical form stores this graph faithfully
(`v(nodes[...],edges[...],regions[...])`) so it round-trips exactly; simple contours may use the terse
`path(...)` form. Region boundaries are directed edge walks using the `+` and `-` operators.

## 7. File format and versioning

- **Extensions**: `.bl` for a Blueprint document (a design/canvas), `.ds` for a design-system document
  (the token DSL, a separate and complementary language). Distinct extensions give editors, language
  servers, and tooling a clean hook.
- **Document shape**: a version stamp `bp:vN`, then any document-level frontmatter (canvas background,
  design-system reference), then `---`, then the Blueprint body.
- **Versioning and backwards compatibility**: every stored document is version-stamped. The interpreter
  keeps per-version decode paths and frozen per-version default tables. Because terseness elides
  defaults, a document's meaning is pinned to its version's defaults, which never change in place. A
  breaking grammar change ships a new version plus a forward migration; additive changes do not. This
  is the discipline that lets the language evolve without ever breaking a stored design.

## 8. The lossless invariant (the contract)

`design -> Blueprint -> design` is structurally identical for every element type and property, and a
re-emission of a decoded document is byte-identical (idempotent). This is enforced by a standing
conformance oracle covering the full element by property cross-product, including the hard cases
(planar graphs with shared edges and holes, boolean and mask parents, components with overrides). A
property that does not round-trip is a bug, not an accepted limitation.

## 9. Related references

- `objects-syntax.md`: how agents emit Blueprint in a reply.
- `knowledge/blueprint/*.md`: the detailed authoring guides (core, layout, paint, text, vectors, lines,
  effects, components, arcs, images, directives).
- The design-system DSL (`.ds`) is documented under the design-system knowledge.
