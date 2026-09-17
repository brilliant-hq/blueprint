---
assumes: blueprint/core, blueprint/layout
dsl: [comp, inst, axes, variant, at, override, slot, lib, projds]
---
# Blueprint Components

Two types, pick by what varies. DEGENERATE (DRY, same shape and only content differs): build it inline, mark the first `comp`, `inst()` the rest, `override()` the content. SET (a catalog of discrete states): build it outside, `comp "Name" axes[state[...]]`, a `variant()` per state, then `inst() at(state(...))`. For both, editing the master propagates to every instance, and editing a variant propagates to every instance in that state. A variant can `inst()` another set (compose an atom into a molecule).

```
-- DEFINE a SET off to the side. axes[axis[labels]] names the axis and its value labels, each variant() child carries the coordinate. props[...] is a byte-identical alias for axes[...].
al(v,g($spacing.lg),pad($spacing.lg)) s(hug,hug) f[($color.surface.container)] rd($radius.lg) "Components" #lib
  comp "Settings Row" axes[accessory[chevron,toggle]] #row
    al(h,y(c),g($spacing.md),pad($spacing.sm,$spacing.md)) variant(accessory(chevron)) s(320,hug) f[($color.surface)] rd($radius.md)
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) slot #row_leading
      t("Label",$font.family,$font.size.md) s(fill,hug) f[($color.text.primary)] #row_title
      svg(icon:caret-right) s(14,14) f[($color.text.disabled)]
    al(h,y(c),g($spacing.md),pad($spacing.sm,$spacing.md)) variant(accessory(toggle)) s(320,hug) f[($color.surface)] rd($radius.md)
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) slot
      t("Label",$font.family,$font.size.md) s(fill,hug) f[($color.text.primary)]
-- USE. at() picks the variant, override(#ref) retargets a #ref'd child, override(#slot) + indented lines fill a slot (the slot element IS the container, the indented lines are its direct children).
al(v,g($spacing.sm),pad($spacing.lg)) s(hug,hug) f[($color.surface.container)] rd($radius.lg) "Settings" #list
  inst(#row) at(accessory(chevron))
    override(#row_title) t("Wi-Fi")
    override(#row_leading) slot
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) f[($color.info)] rd($radius.md)
        svg(icon:wifi-high) s(18,18) f[($color.surface)]
```

Edit a set later by re-declaring axes against the `#ref`: `#row axes[+state[idle,active]]` adds an axis (existing rows take the first value), `#row axes[accessory[+swipe]]` adds a value, `#row axes[accessory->trailing]` renames (keys kept, so `at(trailing(toggle))` still lands), `#row axes[-state]` removes (flags collisions Figma-style, never deletes content). Add a variant by nesting a new `variant(...)` frame under the `#ref`: unseen value labels register automatically, it lands below the existing variants, nothing else moves, and undo reverts the whole edit.

Reconfigure on canvas: `#row_2 at(accessory(toggle))`. `override()` on a non-slot child changes its existing props (locks that category against master edits), and new content needs a `slot`. Fill an EXISTING empty slot later by creating into it, `svg(icon:wifi-high) s(18,18) parent(<slotId>)` (the slot's id from lookup). Before authoring into a slot, read it with `lookup({scope:[<instanceId>], format:"blueprint", expandInstances:true})`: the summary prints only `override(<slotId>) slot`, the expanded form shows the slot's real children.

`override()` targets a master child by the `#ref` assigned when the master was built, by a master-child id (any variant's works), or by exact child name, and lands on that instance's own copy. Give each new element its own fresh `#ref`, because re-using a taken ref keeps the original binding.

Reconfigure a NESTED instance copy by its own id: when a set variant composes another instance (an atom inside a molecule), each outer-instance copy materializes its own copy of that nested instance with its own id. Retarget it with `<copyId> at(axis(value))` (look the id up with `lookup`). This is how you express per-use variant drift (one row's button `primary`, the rest `secondary`), reconfigure the nested copy rather than a raw fill override on it.

## Across canvases

Masters are referenced BY NAME, unique per canvas, in two blessed forms: `inst(Name)` on the same canvas, `inst(Name, canvas(path))` on another (`path` = the identifier you pass as the canvas in tool calls). In one pass a master's `#ref` also resolves across canvases (`inst(#toggle, canvas(Atoms)) "Wi-Fi"`), and from an earlier session you use the name (`inst(Settings Row, canvas(Atoms))`). Quote a name that contains a comma, parens, or quotes, `inst("Card, small")`, while simple multi-word names work either way and round-trip through save. The instance links to its cross-canvas master and follows edits, and the canvas loads on demand (need not be open). Failures are loud: a wrong path or unknown master gives "Component not found", a name matching two masters gives a refusal listing the candidates. Renaming a master onto a name another master already uses on that canvas is refused, so names stay unique. Organizing masters across canvases: `blueprint/libraries`.

## From a library

A project can instance components from a **library**: another published project the current one depends on at a pinned version (declared in `libraries.yaml` at the project root). Add `lib()` with the library's `handle/project` (no `@`), and make `canvas()` the canvas path inside the library:

```
inst(Button, lib(acme/design-kit), canvas(Atoms/Buttons))
```

A local folder library wraps its manifest key in `local()`: `inst(Button, lib(local(my-kit)), canvas(Atoms/Buttons))`. The key is the folder's plain name, written bare even with spaces or parentheses (`lib(local(Material 3 Kit))`), quoted only when it contains a comma, quote, or newline. Older files written as `lib(@acme/design-kit)` or with a bare local key (`lib(my-kit)`) still read fine and rewrite to the current form on the next save. Everything else is a normal instance: `at()` picks a variant, `override()` and slots work as usual, and overrides survive library updates. Library content is read-only from the consuming project, so master edits happen in the library project itself. A library instance renders with the library's own design system by default, add the `projds` token on the instance line to re-theme it with the consuming project's tokens. Adding libraries, updates, and everything else: `reference/libraries`.
