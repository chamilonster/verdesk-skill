---
name: verdesk
description: Use when you need to control the user's desktop (see screen, click, type, read text on screen, run commands) via the Verdesk MCP server. Optimized for language models — returns fewer tokens than a generic screenshot tool. **If you are a large model (Opus or similar), delegate intensive desktop operation to a Haiku/Sonnet subagent and supervise — see "Pick the right model".**
---

# Verdesk — desktop control via MCP

**What this is for you**: an MCP toolbox that lets you see the user's screen and operate their machine. It saves tokens — it does not send you full screenshots every turn, it sends just what changed.

---

## Pick the right model (READ THIS FIRST)

Verdesk works equally with any Claude model, but for **economy and context hygiene** the work should be split:

### If you are Opus (or any large model)

**Do NOT do intensive desktop work yourself.** Intensive = filling forms with multiple fields, clicking through wizards, scraping text from many regions, multi-step task automation. Reasons:

1. Each `look()` / `dry_run` preview returns image bytes that pile up in your context — your reasoning quality degrades after a few hundred KB of crops.
2. Your token price is ~15× a Haiku's. For 40 form fields at ~$0.10 each (Haiku) vs ~$2 each (Opus), the difference is $4 vs $80.
3. The work is mostly mechanical — pick label, derive input rect, dry_run, click, send_keys — exactly what a smaller model can do once it has read this manual.

**Instead, delegate**: spawn a Haiku or Sonnet subagent with a brief that tells it (a) to read this skill first at `~/.claude/skills/verdesk/SKILL.md`, (b) the exact task with data, (c) the report format. Stay in the loop as supervisor:

- Verify with a final `look()` after the subagent reports.
- Spot-check with 1–2 `dry_run` previews on critical fields.
- Handle exceptions and ambiguous cases the subagent flagged.

### If you are Haiku or Sonnet

**You are the operator.** Read this manual carefully, especially:
- "The clicking pattern (READ THIS)" — top-down hierarchy of click tools.
- "Form fields — finding the input from the label" — how to derive the input rect from the OCR label.
- "When filling multiple fields" — re-`look()` between fields, auto-scroll/auto-advance is common.
- "Reporting back" — enumerate every tool call honestly.

Use `dry_run: true` liberally. It is free, and a wrong click on a form can leave the state unrecoverable.

---

## Initial connection (first time)

If the user pasted you a block that looks like this, you are facing a remote setup and must bootstrap yourself:

```
name: <name>
host: <ip>
port: <mcp_port>
control_port: <control_port>
bootstrap_token: <token>
```

If you do NOT see that block (Local mode on the same PC, or the server is already registered in your client), skip this section.

### Bootstrap steps

1. If you do not have this skill active in your client, write this same manual to `~/.claude/skills/verdesk/SKILL.md`. Next turn the skill is ready.

2. Generate a dedicated SSH keypair for this connection (no passphrase):
   ```
   ssh-keygen -t ed25519 -N "" -C "verdesk-<name>" -f ~/.ssh/verdesk_<name>
   ```
   If the file already exists (re-bootstrap), reuse it.

3. Authorize your public key against the server. `POST http://<host>:<control_port>/control/authorize-key`:
   - Header: `X-Verdesk-Auth: <bootstrap_token>` and `Content-Type: application/json`
   - Body: `{"public_key": "<contents of verdesk_<name>.pub>"}`
   - While the call is open, the user sees a popup on their screen to approve. Wait for the response without an aggressive timeout on your end — the user may take minutes.
   - Responses:
     - `200 {"added": true, "ssh_user": "..."}` → approved, save `ssh_user`.
     - `200 {"added": false, "ssh_user": "..."}` → key was already there (idempotent). Save `ssh_user`.
     - `401` → invalid token. Re-check the block.
     - `403` → user rejected. Tell them, do not retry.
     - `400` → public key in invalid format. Regenerate the keypair.

   Accepted algorithms: `ssh-ed25519` (recommended), `ssh-rsa`, `ecdsa-sha2-nistp{256,384,521}`, hardware security keys (`sk-*`).

4. Open an SSH tunnel to the server. Pick a free local port `<localPort>` (can be the same `<port>` or any other high one). In background:
   ```
   ssh -i ~/.ssh/verdesk_<name> -L <localPort>:localhost:<port> -N <ssh_user>@<host>
   ```
   The tunnel must stay alive for the entire session.

5. Register the MCP server in your `.mcp.json` (in the project cwd, or `~/.claude.json` if global):
   ```json
   {
     "mcpServers": {
       "<name>": {
         "type": "http",
         "url": "http://127.0.0.1:<localPort>/mcp"
       }
     }
   }
   ```

6. Tell the user to close and reopen their client so it reloads MCP servers. When they come back, the `<name>` server appears connected.

The user approves only once. Future sessions reuse the same key without a popup.

### Local mode (same PC)

If Verdesk runs on the same machine as you, skip everything above. The user only needs:
```
claude mcp add --transport http verdesk http://127.0.0.1:47802/mcp
```

---

## When to use Verdesk

- **See what is on the user's screen** (any app, game, IDE, terminal, browser).
- **Click, type, or scroll** over their screen.
- **Read plain text** from a screen region without the user copy/pasting.
- **Run a command** in their shell (when working remote and you want to avoid 20 clicks).
- **Visual memory** between turns — what you saw at t-1 vs t-2.

Do not use it for: editing project files (`Read/Write/Edit`), running CI, code search (`Grep`). Verdesk is for **what the user sees on their monitor**, not the codebase.

---

## Workflow

```
Task with screen
       |
       v
  look()  <-- cheap primitive: text + layout, zero pixels
       |
       v
  Know where to act? ----no----> look(zone=...) or refine_cell()
       |
       yes
       v
  Take action (use the highest-level click tool you can)
       |
       v
  Confirm with another look() if doubtful
       |
       v
  Done
```

**Rule 1**: start with `look()` — it is the cheap primitive. Returns visual summary + plain text + layout. Zero pixels by default.

**Rule 2**: if you need more detail, `look(zone=...)` to focus a zone, or `refine_cell(...)` for one cell. **Do not re-capture everything every turn.**

**Rule 3**: to act, use the highest-level primitive available (see "The clicking pattern" below).

**Rule 4**: to read text from screen, use the `text` field that `look()` returns. It arrives flat, ready to reason on.

---

## The clicking pattern (READ THIS)

The biggest source of failed actions in past sessions has been mapping "I see button X around cell C5R2" to an exact `click_at(x, y)`. Don't do the math yourself. Use these helpers, top-down:

### 1. `act_uia(id, action)` — first choice when UIA is available

If `get_capabilities()` returns `has_uia: true` and `list_uia_elements()` shows the target, this is the most robust path. Native semantic action, no pixels involved.

### 2. `click_text("substring")` — when the target is text

After a `look()`, the `text` layer contains collages with substrings. `click_text("Edit")` finds the first match and clicks it. Use `occurrence: "nth"` if there are multiple.

### 3. `click_collage(id)` — when you have a stable collage id

If `look()` already returned a collage and you want to click it, this is the cheapest path. The id is stable within the same frame.

### 4. `click_in_rect(rect, x_pct?, y_pct?, jitter?, dry_run?)` — when you have a precise bbox

You have a `bbox_abs` (from an OCR collage, a layout zone, or a `CellResponse.rect`). Pass it directly.

```json
// Center of the rect:
{"x":842, "y":301, "w":80, "h":24}

// 30% horizontal, 60% vertical inside the rect:
{"x":842, "y":301, "w":80, "h":24, "x_pct":0.3, "y_pct":0.6}
```

Default `jitter: false` — the rect already delimits a precise target (typical OCR bbox).

### 5. `click_in_cell({cell_id, x_pct?, y_pct?, jitter?, dry_run?})` — coarse 12×8 grid target

When you only know roughly which grid cell the target is in. Verdesk resolves the cell rect from the current viewport.

```json
// Click the center of cell (col=5, row=2):
{"cell_id":{"col":5, "row":2}}

// Click at 30% horizontal, 60% vertical inside that cell:
{"cell_id":{"col":5, "row":2}, "x_pct":0.3, "y_pct":0.6}
```

Default `jitter: true` — the 12×8 cell is coarse compared to the target, and a "pixel-perfect center" click is a bot signal. Verdesk adds ±30 px horizontal × ±20 px vertical of human-like noise, clamped to the cell. Pass `jitter: false` if you need pixel-perfect.

### 6. `dry_run: true` — verify before committing (use this!)

Both `click_in_rect` and `click_in_cell` accept `dry_run: true`. The click is NOT executed. Instead you get:

- `{x, y, resolved_rect, preview_size: 80, dry_run: true}` as JSON.
- An 80×80 px WebP crop centered on the target point.
- A **magenta crosshair (+)** drawn on the crop, centered on the exact pixel that would be clicked. There is a small gap at the very center so you can still see the underlying pixel.

If the crosshair is on the wrong thing, recompute and try again. When it looks right, repeat the call with `dry_run: false`. This loop costs almost nothing and rescues many failed actions.

**If you are not 100% confident** that the crosshair is on the right thing — run **one more `dry_run`** with a different offset before committing. Do not commit a click while in doubt; the preview is free and an incorrect click can leave the form in an unrecoverable state.

### 7. `click_at(x, y)` — last resort

Pixel-perfect, no helpers. Use when none of the above fits.

---

## Form fields — finding the input from the label

OCR only sees the **label text**, not the white input rectangle. So when you `look()` a form, the collages give you bboxes for the labels ("Given Name", "City"), but not for the input fields themselves.

Convention used by almost every form (USPTO, Foxit, web forms):

- **Vertical layout** (label on top of input): the input rectangle starts a few px BELOW the label and is typically 25–40 px tall and 1.5–2× as wide as the label text.
- **Horizontal layout** (label on the left of input): the input is just to the right of the label, vertically aligned with it.

So if a label `"Given Name"` has `bbox_abs = {x:533, y:501, w:92, h:12}`, a reasonable guess for the input field's center is:

```
center_x = label.x + 50   (rough — between the label x and a bit into the wider input)
center_y = label.y + 35   (label is ~12 px tall, input starts ~10 px below, input ~20 px tall → click ~35 px below label.y)
```

Always confirm with `dry_run: true`. If the crosshair lands on the label text or on the white margin outside the input, shift the offset 10 px and try again. **Two or three dry_runs are normal** for a form field you have never seen before.

If the form has many similar fields, the offsets typically generalize across them — once you find the right Δy for one field, the rest of the labels in the same column use the same Δy.

---

## When filling multiple fields (READ THIS)

Forms commonly **auto-scroll** or **auto-advance** after each field is filled. Foxit, web forms, and most PDF readers do this. Concretely:

- After you `send_keys` into a field, the cursor may jump to the next field automatically.
- The page may scroll up/down to keep the active field centered.
- Your previous `look()` data is **STALE** after the first `send_keys`.

**Rule**: call `look()` **between every field**. The OCR is cheap (~150–300 ms, ~5K tokens) and re-anchors you to the current state of the form. Do not assume the bbox from `look()` #1 is still valid after `send_keys` #1.

**Pattern**:

```
look() #1                          → identify field 1 label
click_in_rect(..., dry_run=true)   → verify input box
click_in_rect(..., dry_run=false)  → real click
send_keys("value 1")

look() #2                          → re-anchor (page may have moved!)
click_in_rect(..., dry_run=true)   → verify field 2 input box
click_in_rect(..., dry_run=false)
send_keys("value 2")

look() #3                          → re-anchor
... etc ...

look() #final                      → VERIFY: each value appears where expected,
                                     and NO unexpected text appeared anywhere
```

**The final verification look() is mandatory.** Inspect every field you typed in: the text should appear in the position you expected. If you see unexpected text in a field you did NOT intend to fill, it means an auto-advance moved the cursor and your `send_keys` landed somewhere else. Report this and ask the orchestrator how to clean up — do not silently move on.

---

## Tools

### See the screen

| Tool | When |
|------|------|
| `look(zone?, want?, mode?)` | Main primitive. `mode`: `glance` (cheap, default) \| `detail`. `want`: subset of `["visual","text","layout"]` (default `["text","layout"]`). `zone`: `all` \| `rect{x,y,w,h}` \| `cells{ids}` \| `around_collage{id, padding_px?}` \| `around_layout_zone{id}`. |
| `capture(cells?, color_mode?, quality?, ...)` | Per-cell capture (12×8 grid). Returns only deltas. Prefer `look()` for new flows. |
| `refine_cell(cell_id, quality?)` | Re-capture ONE cell at higher quality. |
| `refine_cells(cells, quality?, color_mode?)` | Batched: re-capture N cells in one call. |
| `get_buffer_state(include_thumbnails?)` | What is in the buffer right now — metadata only. |
| `get_capabilities()` | Surface flags: `has_capture`, `has_input`, `has_text_layer`, `has_uia`, `has_target_switch`, etc. Call before planning a flow. |
| `extract_text(target)` | Read PLAIN TEXT from: `elements{ids}` (DOM/UIA innerText) \| `cells{selector}` (from capture) \| `region{rect}` (viewport rect). Plain text, not pixels. |
| `compare_cells(cell_a, cell_b)` | Hamming pHash/dHash distance between two cells in the buffer. |
| `compare_cells_matrix(cells)` | Triangular i<j of N cells. N*(N-1)/2 pairs. Minimum 2. |
| `query_buffer(phash, threshold?)` | Visual associative memory over the active buffer. |

### Act on the screen

| Tool | When |
|------|------|
| `act_uia(id, action)` | Semantic action on a UI Automation element. `action`: `{kind, value?}` — `kind`: `invoke` \| `set_value` (+`value`) \| `toggle` \| `select` \| `expand` \| `collapse`. `id` is an `auto_NNN` from `list_uia_elements`. |
| `list_uia_elements(visible_only?, max_depth?)` | UIA tree inventory. Desktop only. |
| `wait_for_uia(id, condition, timeout_ms?)` | Poll until an element matches `enabled` \| `visible` \| `name_contains{text}` \| `value_equals{text}` \| `toggle_state{on}`. |
| `wait_for_uia_property_change(id, property, timeout_ms?)` | Event-driven (no polling). Properties: `value` \| `toggle_state` \| `enabled` \| `offscreen` \| `name`. |
| `click_text(query, occurrence?)` | Click on a substring on screen. `occurrence`: `first` \| `last` \| `nth` \| `all`. |
| `click_collage(id)` | Click on a stable collage from the last `look()`. |
| `click_in_rect({x,y,w,h, x_pct?, y_pct?, jitter?, dry_run?})` | **Read "The clicking pattern".** Precise rect target, with optional dry_run preview. |
| `click_in_cell({cell_id, x_pct?, y_pct?, jitter?, dry_run?})` | **Read "The clicking pattern".** Coarse 12×8 cell target, default human jitter, with optional dry_run preview. |
| `click_at(x, y)` · `send_keys(text)` · `press_key(key, ctrl?, alt?, shift?, win?)` | Low level: coords, type text, key combo. |
| `scroll(direction, amount_px)` | Scroll the viewport. |
| `focus_window(hwnd_hex)` | Bring a window to the front before sending input. |
| `focus_zone(zone)` · `unfocus_zone()` · `get_focus_zone()` | Set / release / read the focused zone. `look()` without `zone` starts from the focused zone until you release it. |

### Surface (capture target)

| Tool | When |
|------|------|
| `list_surfaces()` | Enumerate monitors + visible top-level windows. Each entry has a `spec` ready for `set_surface`. The one with `current: true` is the active one. |
| `set_surface(target)` | Switch the target. Spec: `monitor` \| `monitor:N` \| `window:0xHWND` \| `window-title:"text"`. Invalidates buffer + UIA inventory. |

### Buffer lifecycle

| Tool | When |
|------|------|
| `force_reset()` | Hard reset of the active buffer (archives to history with `reason: force`). |
| `pin_buffer()` · `unpin_buffer()` | Veto / lift the next automatic reset. Useful before an expected big change. |

### Visual memory across turns

| Tool | When |
|------|------|
| `list_history(reason?, url_contains?, limit?, ...)` | Archived snapshots with their reset reason. |
| `get_historical_snapshot(snapshot_id, include_thumbnails?)` | Full snapshot by id. |
| `query_history(phash, threshold?)` | "Did I see this before?" Cross-session associative search. |

### Modulation profiles

| Tool | When |
|------|------|
| `list_profiles(target_match?, min_rating?, creator_kind?)` | Saved recipes. |
| `load_profile(id? \| target_match?)` | Best profile for a target. |
| `save_profile(target_type, target_match, params, creator, ...)` | Save a recipe when you find a good combination. |
| `update_profile(id, name?, params?)` | Overwrite name + params keeping rating/creator/created_at. |
| `rate_profile(id, rating?)` | Report how well it performed (0.0–1.0). |
| `delete_profile(id)` | Delete a profile. |

### User's shell

| Tool | When |
|------|------|
| `run_command(command, cwd?, timeout_ms?)` | Run a command on the user's PC. Returns stdout/stderr/exit_code. |

---

## Common gotchas

- **You see something but `look()` does not** → wrong surface. `list_surfaces()` + `set_surface(...)` to the right monitor/window. Verdesk's capture is scoped to one surface at a time.
- **`look()` returns empty `text`** → either the surface is blank, or you need `mode: "detail"` for finer OCR. Try once more before giving up.
- **`click_text` matches nothing** → the text on screen is not literally what you typed (case mismatch, special characters, hidden whitespace). Try a shorter or unique substring.
- **Click landed on the wrong thing** → use `dry_run: true` with `click_in_rect` or `click_in_cell` next time. The crosshair tells you immediately.
- **Action looks like nothing happened** → some apps need the window focused first. `focus_window(hwnd_hex)` before the click.

---

## Reporting back

When you finish a task, your report MUST include a **literal enumeration** of every MCP tool call you made, in order:

```
tool_calls:
  1. look()
  2. click_in_rect(x=842, y=301, w=80, h=24, dry_run=true)
  3. click_in_rect(x=842, y=301, w=80, h=24, dry_run=false)
  4. send_keys("Camilo")
  5. look()
  6. click_in_rect(...)
  ...
tool_calls_total: <count the lines above, do not guess>
```

Do not write "one attempt", "a few tries", "several look()s" — write the literal list. Counting from memory or estimating is forbidden — your harness can give you the exact list if you don't remember.

Also include:
- One line per field: `<Field>: dry_runs=<n>, final_coords=(x,y), typed="<value>", verified_in_final_look=true|false`.
- Any field where the dry_run preview was ambiguous and you had to fall back on heuristics.
- Any unexpected behavior you saw (auto-advance, auto-scroll, popup, error message).
- The output of the final verification `look()` for each typed value — explicitly: "saw 'Camilo' at y=371, expected, OK" or "did NOT see 'Brossard' anywhere, abort".

Bad: "One attempt." — hides 12 internal tool calls and prevents the orchestrator from auditing the work.
Good: full enumeration as above.

## Tone with the user

The Verdesk user wants you to **do the task**, not explain how. When you achieve what they asked, tell them what you did in one sentence + the result. Do not detail every tool call unless they ask.
