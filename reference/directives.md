---
assumes: blueprint/core
dsl: [parent, before, after, clone, replace, ungroup, undo]
---
# Blueprint Directives

Directives edit elements that already exist. They mix freely with create
and modify lines in one call; target by 16-char `id` or session `#ref`.

A `#pricing` card from an earlier call, revised in one pass. `--` lines
are notes; an inline `// label` snapshots an undo checkpoint after its
line runs (before any indented children).

```
before(#pricing) fr s(360,480) f[(radial($primary.soft,$color.surface))] rd($radius.xl) "Glow"
-- before() on the NEW element places it earlier in z-order, behind #pricing

al(h,pad($spacing.sm)) after(#logo) parent(#nav) "Search"
-- after(#sibling) places the element right AFTER that sibling (the mirror
-- of before()). after() the LAST child appends to the end. The element id
-- must LEAD the line: `#icon after(#label)`, never `after(#label) #icon`.

#icon after(#label)
-- on an existing element, after() reorders it to sit just after #label
-- (reparenting into #label's parent if needed)

#popular parent(#pricing)
-- parent() reparents an existing element; its on-screen position holds
al(v,g($spacing.sm)) s(fill,hug) parent(#pricing) "New row"
-- on a CREATE, parent() puts the new element inside #pricing instead of as a sibling

ungroup(#legacy_header)
-- ungroup() dissolves a frame/group, lifting its children into the parent

replace(#cta) al(h,x(c),y(c),g($spacing.sm),pad($spacing.sm,$spacing.lg)) s(fill,hug) f[($color.primary)] rd($radius.md) "Buy"
-- replace() deletes #cta, inserts the new element at its exact position

delete(#placeholder)  // structure revised
-- delete() removes an element and its children

clone(#pricing) p(400,0) ds(, theme(dark)) "Pricing Dark" #pricing_dark
  #plan_name t("Pro")  // dark variant added
-- clone() deep-copies; clone-line props override its root, indented
-- child lines (leading #ref) retarget descendants. The // on that last
-- child checkpoints the finished clone.
```

A `// label` snapshots the session undo stack. Later, in any call this
session, jump between snapshots:

```
undo("structure revised")    -- rewind: drops #pricing_dark, back to the checkpoint
redo("dark variant added")   -- changed your mind: replay forward, it returns
```

Labels are interchangeable: `undo("dark variant added")`,
`undo(dark_variant_added)`, `undo(#dark_variant_added)`.
