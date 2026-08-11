---
assumes: blueprint/core, blueprint/layout
dsl: [comp, inst, axes, variant, at, override, slot]
---
# Blueprint Components

Two types of components, pick by what varies:

|           | degenerate                              | set                                          |
|-----------|-----------------------------------------|----------------------------------------------|
| mindset   | DRY                                     | library                                      |
| use when  | same shape, only content differs        | discrete states |
| placement | *inline*, first-as-master                 | *outside*, first-as-instance |
| build     | mark the first `comp`, `inst()` the rest, `override()` content | `comp … axes[state[…]]`, a `variant()` per state, then `inst() at(state(…))` |

For both, editing the master propagates to every instance. Editing a variant, to every instance in that state. A variant can `inst()` another set.

```
-- DEFINE off to the side. A small reusable set...
al(v,g($spacing.lg),pad($spacing.lg)) s(hug,hug) f[($color.surface.container)] rd($radius.lg) "iOS Components" #lib
  comp "Toggle" axes[state[on,off]] #toggle
    al(h,x(e),y(c),pad($spacing.xs)) variant(state(on)) s(51,31) f[($color.success)] rd($radius.full)
      al(pad($spacing.none)) s(23,23) f[($color.surface)] rd($radius.full)
    al(h,x(s),y(c),pad($spacing.xs)) variant(state(off)) s(51,31) f[($color.surface.container.high)] rd($radius.full)
      al(pad($spacing.none)) s(23,23) f[($color.surface)] rd($radius.full)
  -- ...nested inside a bigger one. accessory[] is the axis; each variant is a row.
  comp "Settings Row" axes[accessory[chevron,toggle]] #row
    al(h,y(c),g($spacing.md),pad($spacing.sm,$spacing.md)) variant(accessory(chevron)) s(320,hug) f[($color.surface)] rd($radius.md)
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) slot #row_leading              -- slot: instance fills it
      t("Label",$font.family,$font.size.md) s(fill,hug) f[($color.text.primary)] #row_title   -- ref: overridable
      svg(icon:caret-right) s(14,14) f[($color.text.disabled)]
    al(h,y(c),g($spacing.md),pad($spacing.sm,$spacing.md)) variant(accessory(toggle)) s(320,hug) f[($color.surface)] rd($radius.md)
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) slot
      t("Label",$font.family,$font.size.md) s(fill,hug) f[($color.text.primary)]
      inst(#toggle) at(state(on))                                                -- compose: this row (organism) instances the Toggle (molecule)

-- USE. at() picks the variant; override() retargets a #ref'd child; override(#slot) slot + indented fills a slot.
al(v,g($spacing.sm),pad($spacing.lg)) s(hug,hug) f[($color.surface.container)] rd($radius.lg) "Settings" #list
  inst(#row) at(accessory(chevron))
    override(#row_title) t("Wi-Fi")
    override(#row_leading) slot
      al(h,x(c),y(c),pad($spacing.none)) s(29,29) f[($color.info)] rd($radius.md)
        svg(icon:wifi-high) s(18,18) f[($color.surface)]

-- EDIT later by re-declaring axes against the #ref; variants + instances follow.
#row axes[+state[idle,active]]       -- + adds an axis (existing rows get the first value)
#row axes[accessory[+swipe]]         -- + adds a value
#row axes[accessory->trailing]       -- -> renames, keys kept (at(trailing(toggle)) still lands)
#row axes[-state]                    -- - removes (flags collisions Figma-style, never deletes content)
-- ADD A VARIANT: nest it under the #ref (unseen value labels register automatically).
-- It lands below the existing variants; nothing else moves. Undo reverts the whole edit.
#row
  al(h,y(c),g($spacing.md),pad($spacing.sm,$spacing.md)) variant(accessory(swipe)) s(320,hug) f[($color.surface)] rd($radius.md)
    t("Label",$font.family,$font.size.md) s(fill,hug) f[($color.text.primary)]
```

Reconfigure on canvas: `#row_2 at(accessory(toggle))`; `override()` on a non-slot child changes its existing props (locks that category vs master edits); new content needs a `slot`.

Reconfigure a NESTED instance copy the same way, by its own id: when a set variant composes another instance (an atom inside a molecule), each outer-instance copy materializes its own copy of that nested instance with its own id. Retarget it with `<copyId> at(axis(value))` (look the id up with `lookup`). This is the right way to express per-use variant drift (e.g. one row's button is `primary`, the rest `secondary`) — reconfigure the nested copy, not a raw fill override on it.

`override()` targets a master child, by the `#ref` assigned when the master was built, by a master-child id (any variant's works), or by exact child name, and lands on that instance's own copy. Give each new element its own fresh `#ref`; re-using a taken ref keeps the original binding.

## Across canvases

Masters are referenced BY NAME, unique per canvas — two blessed forms: `inst(Name)` on the same canvas, `inst(Name, canvas(path))` on another (`path` = the identifier you pass as the canvas in tool calls). In one pass a master's `#ref` also resolves across canvases (`inst(#toggle, canvas(Atoms)) "Wi-Fi"`); from an earlier session, use the name (`inst(Settings Row, canvas(Atoms))`). Quote a name that contains a comma, parens, or quotes — `inst("Card, small")`; simple multi-word names work either way, and names round-trip through save. The instance links to its cross-canvas master and follows edits; the canvas loads on demand (need not be open). Failures are loud, never silent: a wrong path or unknown master → "Component not found"; a name matching two masters → a refusal listing the candidates. Names stay unique because renaming a master onto a name another master already uses on that canvas is refused (you pick another). Full structure: `blueprint/libraries`.