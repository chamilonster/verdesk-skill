# Verdesk — desktop (windows & monitors)

`look()` first — on-screen text as collages + layout, almost no pixels. Then act on a UIA control, a piece of visible text, or a box the engine hands you. **`focus_window` before every input burst** — focus drifts.

## Gotchas (read first)
- **Typed a URL / pressed keys, nothing changed.** → The wrong window has OS focus (`set_view_target` ≠ focus; focus drifted). `focus_window(hwnd)` → `look()` to confirm the right window is front → retry. Never input blind after a gap.
- **Click hit the wrong thing.** → You aimed by eye. Use `dry_run:true`; the crosshair shows the exact pixel before you commit.
- **`look()` shows the wrong app / a black frame.** → Wrong surface. `list_surfaces()` + `set_view_target(...)`.
- **`click_text` matched nothing.** → On-screen text isn't literally what you typed (case/whitespace). Use a shorter unique substring, or `look(mode:"detail")` then `click_in_rect` with `dry_run`.

## Coordinate model (read once)
On a window surface every `(x,y)` is **relative to that window** — `(0,0)` = its top-left. These are your **fixed cords**: stable. Verdesk converts them to absolute screen (real cords) to send input and keeps them valid even if the window moves — via the automatic anchor. Two different things, do not confuse them (the #1 cause of "nothing happened"): **`set_view_target`** = what you SEE + the coordinate frame; **`focus_window`** = OS keyboard focus = where input LANDS.

## Pick what you see — context
- `list_surfaces()` → monitors + windows, each with a `spec`.
- `set_view_target(spec)` → `monitor` | `monitor:N` | `window:0xHWND` | `window-title:"text"`. Switches what you SEE + the frame. Does NOT move keyboard focus. Invalidates the buffer + UIA inventory.

## See — `look(zone?, want?, mode?)`
Returns: `frame` (viewport/grid/title/freshness) · `text[]` collages `{id, kind, text, bbox_abs}` (the box you feed to `click_in_rect`/`learn_button`) · `layout[]` zones · `macros[]` (Pro: trains matching the screen NOW, closest first — autocomplete for tasks).
- `want` = subset of `["text","layout","visual"]` (default `["text","layout"]`, no pixels; add `"visual"` only to see shape).
- `mode` = `glance` (cheap default) | `detail` (finer text).
- `zone` = `{kind:"all"}` · `{kind:"rect",rect:{x,y,w,h}}` · `{kind:"cells",ids:[…]}` · `{kind:"around_collage",id}` · `{kind:"around_layout_zone",id}` · `{kind:"selector",css}` (resolves ANY element's bbox — even a non-interactive `<div>`/drop-zone).
- **Why not read pixels for text:** images come light (good for shape: icons, layout). For TEXT use the text layer or `read_text` — at medium res a model hallucinates words.

## Read exact text — `read_text(target)`
Deterministic plain text. `target` is exactly one of `{region:{rect}}` · `{elements:{ids}}` (UIA innerText) · `{cells:{selector}}`. Prefer over reading an image. The window title is already in `look().frame.title`. OCR can misread long numbers/codes (`8.511.708`→`8.511.7e8`, `O`↔`0`) — when a value MUST be exact, `screenshot` it zoomed and read the digits off the enlarged image.

## See one thing as an image — `screenshot({target|rect, max_dim?})`
One rendered crop for what the text layer can't give: an icon (by shape), a chart, a custom-drawn region. `max_dim` = zoom (upscales up to 4×). Monitor-agnostic — Verdesk's own capture, not an OS grab.

## UIA — semantic controls (no pixels, survives moves)
- `list_uia_elements()` → `auto_NNN` ids.
- `act_uia({id, action})` — `action.kind` ∈ `invoke | set_value(+value) | toggle | select | expand | collapse`. Its ids also feed clicks & waits. ex: `act_uia({id:"auto_07", action:{kind:"invoke"}})`.

## Act — the click ladder (top→down, most robust first)
1. `act_uia({id, action})` — semantic UIA when `get_capabilities().has_uia`. No pixels → survives moves.
2. `click_text(query, occurrence?)` — a visible substring from `look().text`. `occurrence` ∈ `first|last|nth|all`. ex: `click_text("Pinceles")`.
3. `click_collage(id)` — a collage from the last `look()`.
4. `click_in_rect({x,y,w,h, x_pct?, y_pct?, button?, dry_run?})` — a precise box. Center by default. ex: `click_in_rect({x:387,y:100,w:39,h:8})`.
5. `click_in_cell({cell_id:{col,row}, …})` — a coarse 12×8 cell.
6. `click_at({x,y, button?})` — pixel-perfect, last resort.

`button` ∈ `left`(default)|`right`(context menu)|`middle`. **`dry_run:true`** on `click_in_rect`/`click_in_cell` fires NO click — returns `{x,y,resolved_rect}` + an 80×80 crop with a magenta crosshair on the exact pixel. Free; use whenever unsure, then repeat with `dry_run:false`.

**Other input:** `type_text(text)` · `press_key({key, ctrl?, alt?, shift?, win?})` · `scroll({direction, amount_px})` · `drag_path({points:[{x,y}…], button?, hold_ms?})` (press→move→release; drawing/sliders/drag&drop — points are viewport coords).

## Window & coordinates
- `focus_window(hwnd_hex)` → OS keyboard focus. **Re-call before every input burst** (focus drifts).
- `get_window_geometry()` → `{x,y,w,h, client, monitor, minimized, corners}`.
- `set_window_size({width,height})` → resizes the client area; no move / no focus steal. Use to restore the size a train was recorded at.
- `set_look_zone(zone)` / `clear_look_zone()` / `get_look_zone()` → a capture default for bare `look()` (NOT keyboard focus).

## Wait on a condition, not a clock
- `wait({ms})` — sleep (cap 60000); only to debug.
- `wait_for_uia({id, condition, timeout_ms?})` — `condition.kind` ∈ `exists | enabled | visible | value_contains{substring} | value_equals{value} | toggle_state{state}`.
- `wait_for_uia_property_change({id, property, timeout_ms?})` — `property` ∈ `value | toggle_state | enabled | offscreen | name`.

## Advanced — buffer, memory, profiles, low-level capture (rarely needed)
The visual buffer auto-resets on a big screen change (title/URL/readyState, >40% DOM, >60% cells); you rarely manage it — check `look().frame.freshness`. If you must:
- Buffer/memory: `get_buffer_state` · `query_buffer(phash)` · `force_reset` · `pin_buffer`/`unpin_buffer` · `list_history` · `get_historical_snapshot` · `query_history(phash)`.
- Profiles (saved capture recipes): `list_profiles` · `load_profile` · `save_profile` · `update_profile` · `rate_profile` · `delete_profile`.
- Low-level 12×8 capture: `capture(cells)` · `refine_cell`/`refine_cells` · `compare_cells`/`compare_cells_matrix`. Prefer `look()`.
