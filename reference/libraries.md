---
assumes: blueprint/components
---
# Component Libraries (multi-canvas projects)

Treat a big project like a codebase. Each canvas is a module, its components
are that module's exports, and a component reference is the API between
canvases. Split by layer so a screen never redefines a primitive:

- **Atoms**: the smallest reusable pieces (Button, Toggle, Input, Badge).
- **Molecules**: atoms composed into units (Settings Row, Card, List Item).
  A molecule variant can `inst()` an atom.
- **Screens**: full pages that `inst()` molecules, never redraw them.

Give each layer its own canvas (or a folder of canvases) and name it for the
layer. Masters live once, on the canvas they belong to; everything else
instances them.

## Consume across canvases

`inst(Name, canvas(path))` places an instance of a master on another canvas;
`inst(Name)` does it on the same one. Names ARE the reference, unique per canvas
(enforced — renaming a master onto a name another master already uses is
refused), so a name is unambiguous. In one authoring pass a `#ref` also resolves
across canvases (`inst(#toggle, canvas(Atoms))`); from an earlier session, use
the name (`inst(Settings Row, canvas(Atoms))`). Quote a name with a comma,
parens, or quotes. The master canvas loads on demand, and master edits propagate
to every instance everywhere. A name matching two masters refuses loudly listing
the candidates. Full form and failure shapes: `blueprint/components`.

## Safe today vs coming

Building a library and instancing it across canvases is solid today. What is
NOT yet supported is REORGANIZING existing masters across canvases.

WARNING: do not cut/paste a master from one canvas to another. It breaks every
instance that referenced it (the reference does not follow the move). Decide a
master's home canvas up front and build it there. If you truly must relocate
one, rebuild the master on the target canvas and re-instance, rather than
moving the original.
