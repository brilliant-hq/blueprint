---
assumes: blueprint/core
dsl: [execute_commands, select_elements, use_mask, detach_component, push_overrides, import_figma, create_canvas, boolean]
---
# Blueprint Commands

`execute_commands` is only for operations the Blueprint DSL cannot
express: alignment, distribution, selection, canvas management, view
toggles, undo/redo. Creation, deletion, property changes, z-order, flip,
group/ungroup, and reparenting are all DSL operations, never commands.

```json
{ "canvasId": "...", "commands": [{ "commandId": "...", "elementIds": [...], "params": {...} }] }
```

Commands run sequentially; pass `previewIds` + `previewScale` for a PNG.

## Commands

- **Selection**: `select_elements`, `deselect_all`
- **Align** (2+): `align_left/right/top/bottom`, `align_horizontally`, `align_vertically`, plus `center_horizontally` / `center_vertically` (center on canvas)
- **Distribute** (3+): `distribute_horizontally`, `distribute_vertically`
- **Boolean**: `boolean_union`, `boolean_subtract`, `boolean_intersect`, `boolean_exclude`
- **Mask**: `use_mask` builds a clipping mask from the selection
- **Components**: `detach_component`, `reset_component_instance_overrides`, `push_overrides_to_master`, `go_to_master_component`
- **Import**: `import_figma` (`{figmaUrl}`)
- **Canvas**: `create_canvas` (`{fullPath}`), `get_canvases`, `rename_canvas`, `delete_canvas`, `duplicate_canvas`, `create_folder`, `delete_folder`, `create_structure`
- **Background**: `set_background_color` (`{value}`), `toggle_background`, `toggle_whiteboard`, `toggle_blackboard`
- **Appearance** (no `elementIds`): `toggle_dark_mode` flips the app between light and dark; `set_theme_follow_system` makes appearance track the OS setting
- **Keybindings**: `list_keybindings`, `set_keybinding`
- **Provider keys**: `set_anthropic_api_key` and the `_openai_` / `_google_` / `_openrouter_` variants
- **View toggles** (no `elementIds`): `toggle_pixel_grid`, `toggle_snap_to_pixel_grid`, `toggle_rulers`, `toggle_layout_grids`, `toggle_snap_guides`, `toggle_dimension_labels`, `toggle_presentation_mode`, `toggle_ui`
- **Element toggle**: `toggle_constrain_proportions` (needs `elementIds`)
- **Suggest a setting**: `suggest_setting_change` (see below)

## Suggesting a configuration change

When the user's design-system intent is unclear, SUGGEST rather than
assume. `suggest_setting_change` shows the user a card they accept or
decline; you keep working and their decision arrives as a later message.
Params:

- `setting`: `designSystem` (this chat's DS selection) or
  `dsEditPermission` (whether you may author `ds_file` changes).
- `options`: 1 to 3 proposed values. For `designSystem`: `none`, `new`,
  or `explicit:<brand>` (a brand that exists in the project). For
  `dsEditPermission`: `allowed` or `denied`.
- `reason`: one short line shown verbatim on the card.

The user always also gets a "Keep current" button. One suggestion is
pending at a time. Accepting applies the change exactly as if the user
had picked it in the design-system picker.

```json
{ "canvasId": "...", "commands": [{ "commandId": "suggest_setting_change",
  "params": { "setting": "designSystem", "options": ["explicit:acme", "none"],
              "reason": "These screens match the acme brand." } }] }
```

## Undo / redo

Each AI session has its own undo stack: `undo` reverts only your last
action, never the user's work (their Cmd+Z likewise skips yours). For
multi-step rollback prefer the inline `undo("label")` directive over N
bare `undo` calls (see `blueprint/directives`).
