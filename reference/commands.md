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

A canvas that failed to load refuses every mutating command with an error naming the canvas (its saves are blocked, so edits there would be lost). Read-only commands like `copy` still work, and `delete_canvas` stays available so a corrupt canvas can be removed.

Commands run sequentially; pass `previewIds` + `previewScale` for a PNG.

Element-operation commands act only on the `elementIds` (or `parts`) you pass: they never read the app's live selection. **An element command called with no targets is refused, never silently succeeded**: if it cannot name an element it cannot change anything, so it says so instead of reporting success. Only commands that genuinely take no target run id-less (canvas background and board, `clear_elements`, undo/redo/paste, view and tool toggles, zoom, canvas/folder management, design-system and library acts). `set_stroke_dash` on an element that has no stroke likewise refuses (add a stroke first).

`elementIds` belongs on the command, as shown above. If you put the same key beside `canvasId` (it then applies to every command that names none of its own) or inside `params`, it is still read. A target id under a DIFFERENT key, like `params.value`, is refused with an error naming the key to use: it is never guessed at.

**Params are read, never guessed or defaulted.** A param key a command cannot consume is refused by name, and the error names the keys it does read: `move_elements {deltaX, deltaY}` is refused rather than applied as a move of zero (it takes `{dx, dy}`; `skew_elements` takes `{skewX, skewY}`). A missing REQUIRED param is refused the same way, so a call that would change nothing never comes back green. Naming one axis is fine, the other stays zero.

## Commands

- **Selection**: `select_elements`, `deselect_all`
- **Align** (2+): `align_left/right/top/bottom`, `align_horizontally`, `align_vertically`, plus `center_horizontally` / `center_vertically`. Align moves the targets onto the bounds they SHARE, so it needs 2 or more elements with the same parent: aimed at a single element it refuses (there is nothing to align to) and points at the centering command for that axis. To place ONE element relative to its parent use `center_horizontally` / `center_vertically`, or set the position outright (`p(x,y)` in the Blueprint DSL). Centering an element that lives **inside a frame** centers it within that frame, on any canvas. Centering a **top-level** element centers it in the visible viewport, which exists only for the canvas currently on screen: aimed at any other canvas it refuses and says so, so give top-level elements an explicit position instead (`move_elements {dx, dy}`, or `p(x,y)` in the Blueprint DSL).
- **Distribute** (3+): `distribute_horizontally`, `distribute_vertically`. Below 3 targets in one parent, or with the elements already packed edge to edge, it refuses and says which: it never reports success over a canvas it could not change.
- **Boolean**: `boolean_union`, `boolean_subtract`, `boolean_intersect`, `boolean_exclude`
- **Mask**: `use_mask` builds a clipping mask from the selection
- **Components**: `detach_component`, `reset_component_instance_overrides`, `push_overrides_to_master`, `go_to_master_component`
- **Import**: `import_figma` (`{figmaUrl}`)
- **Canvas**: `create_canvas` (`{fullPath}` — with or without `.bl`, one extension is minted either way), `get_canvases`, `rename_canvas` (`{newName}`), `delete_canvas` (deletes the call's `canvasId`; pass `{fullPath}` to name a different target — a named target that doesn't resolve refuses, it never falls back to `canvasId`), `duplicate_canvas`, `create_folder`, `delete_folder`, `create_structure`
- **Open a file**: `open_file` (`{value: "<repo-relative path>"}`) opens that file as the active file, exactly like clicking it in the file explorer — `.bl` canvases open on the canvas, text/markdown files open in the editor; a bad path errors
- **Background**: `set_background_color` (`{value}`), `toggle_background`, `toggle_whiteboard`, `toggle_blackboard`
- **Appearance** (no `elementIds`): `toggle_dark_mode` flips the app between light and dark; `set_theme_follow_system` makes appearance track the OS setting
- **Keybindings**: `list_keybindings`, `set_keybinding`
- **Provider keys**: `set_anthropic_api_key` and the `_openai_` / `_google_` / `_openrouter_` variants. Storing a FRESH key succeeds directly. Overwriting an EXISTING key shows the user an in-app confirmation card and returns without storing; if the user approves, the result message tells you to call the same command again (that one follow-up call stores the key), and if they decline the existing key is kept. You cannot approve on the user's behalf.
- **View toggles** (no `elementIds`): `toggle_pixel_grid`, `toggle_snap_to_pixel_grid`, `toggle_rulers`, `toggle_layout_grids`, `toggle_snap_guides`, `toggle_dimension_labels`, `toggle_presentation_mode`, `toggle_ui`
- **Element toggle**: `toggle_constrain_proportions` (needs `elementIds`)
- **Libraries** (no `canvasId`/`elementIds`; each answers with an honest verdict — server refusals verbatim):
  - Producer (the open project must be synced to Brilliant cloud): `mark_project_as_library` / `unmark_project_as_library` flip the library flag (unmarking keeps existing releases resolvable); `create_library_release` (`{version?, notes?}`) checkpoints and binds a plain-semver version — omit `version` for latest-patch+1 (1.0.0 when none); `notes` becomes the changelog entry.
  - Consumer (writes `libraries.yaml` at the project root): `add_library` (`{library: "@handle/project", version?}` — validates the library flag + the release; default = latest), `update_library` (same params; bumps the pin, default latest), `remove_library` (`{library}` — instances go missing-library loud, never stripped), `add_local_library` (`{libraryName, path}` — a local folder as a live-updating dependency, folder-backed sessions only).
- **Cover** (the one image a project presents itself with, on its tile and cards — load `design/covers` before designing one; each answers with an honest verdict):
  - `set_project_cover` (`{canvasId, elementId?}` — `canvasId` is required and names the canvas the cover lives on; `elementId` names a FRAME on it, and omitting it uses the whole canvas). Replaces any existing cover: there is one cover per project.
  - `remove_project_cover` (no params) — the project falls back to its canvas mosaic.
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
