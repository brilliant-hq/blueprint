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

## Reorganizing masters across canvases

You can relocate a master to another canvas with `parent(#master, canvas(path))`
(the same target grammar as `inst(...)`, applied to the master itself). Every
instance that referenced it follows automatically: the move re-points them,
same-canvas instances become cross-canvas, and the whole relocation is one
undoable step. If any consumer canvas cannot be updated, the move is refused and
nothing changes, so you never end up with dangling references.

Two refusals to know about:
- Move the master ROOT itself. Moving a plain frame that merely CONTAINS a
  master is refused, because it would strand the nested master's instances;
  extract the master directly instead.
- Do not pair the move with a delete of that master in the same call.
