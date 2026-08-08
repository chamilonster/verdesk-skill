# Verdesk — trains (record once, replay cheap) + objects + run

The first move of any multi-step task is **`suggest_macros()`**. A sequence that works becomes a **train** — the saved, self-verifying version you replay cheaply, even with a smaller model. A batch you didn't bother to save is `run_steps`.

## Gotchas (read first)
- **A replay failed.** → `playbook_replay` returns `{ok:false, failed_step, failed_tool, error, blackbox}`. Read it + open the `blackbox` PNG. A coord step that drifted → re-record or `playbook_edit_step`; prefer `click_button` objects for robustness.
- **`find_button` / `track_object` returns `{found:false}`.** → Not on this surface, off the viewport, or appearance changed past threshold. re-`look()`; `object_thumbnail` to recall its shape; relearn; or `click_text` if it has text.
- **`run_command` errored / refused.** → OFF by default. Ask the user to enable Settings → Advanced → "Permitir run_command", or use `open_run` (visible).
- **A `(Pro)` tool errors on a free server.** → trains/objects/linked-objects need Pro. Basic see/click/read/run work in free.

## Batch a live sequence — `run_steps({steps:[{tool,args},…]})`
When you already know the exact mechanical steps AND the state they run in (fill these 4 fields, then click Submit), send them as ONE call instead of N round-trips. Verdesk runs them in order, reads each step's F1 outcome, and **STOPS at the first that didn't land** → `{ok:false, failed_step, tool, reason, result, results}` with that step's evidence + crosshair crop. All land → `{ok:true, executed, total, results}`. A successful batch is also recorded — you can `playbook_save` it. **Only batch deterministic steps inside a state you already see** — you can't batch across an unknown transition (page 2 of a wizard you haven't looked at). Supported tools: `navigate, browser_click, browser_fill, browser_type, browser_transfer, browser_get_value, browser_select_tab, press_key, type_text, scroll, wait, click_at, click_in_rect, click_text, drag_path, act_uia, focus_window, set_window_size, clipboard, set_view_target, run_command, click_button`.

## Trains (Pro)
- `suggest_macros()` → trains whose recorded start-screen matches what's on screen NOW, closest first (`match_score` 0 = identical). Read-only. If one fits → `playbook_replay`; else do the task once → `playbook_save`.
- `playbook_record({task, description?})` — start, THEN do the task. Every mutating action is logged with its real delay.
- `playbook_save({validate?})` — persist after success; re-recording overwrites. `validate:true` stores a success signature. Any replay that completes every step without error bumps `success_count`.
- `playbook_recall({task})` → `{steps:[{tool, args, delay_before_ms, note, anchor}], params}`. `params` = the function signature.
- `playbook_replay({task, args?})` — fills `{slots}`; honors delays. `click_button` steps re-resolve by pixel each run (survive move/resize/relayout); coord steps replay window-relative to the anchor. **Health-check:** before a full replay it confirms the first anchored target is on screen — missing → `{ok:false, stale:true, checked_step}` WITHOUT acting. **Black box:** a PNG per input step under `<config>/verdesk/blackbox/<slug>/` (never sent to you, zero tokens); a failed step's result carries its `blackbox` path.
- `playbook_set_delays({task, delays?|uniform_ms?|scale?})` · `playbook_edit_step({task, step, args?, delay_before_ms?, tool?})` (edit one 0-based step in place) · `playbook_annotate({task, notes:[{step, note}]})` · `playbook_list()` · `playbook_delete({task})`.
- **Autonomy (supervised → trusted):** each train carries `success_count`, `stars` (0-5), `autonomy` (`supervised`|`trusted`), `reversibility` (`read_only`|`benign`|`mutating`|`irreversible`). Fresh = supervised (watch it). ≥3 clean shots → trusted (unattended). A trusted train that breaks drifts back to supervised until repaired. Give even a trusted train a look before an `irreversible` step (a `run_command` can't be undone).
- **Self-healing:** a train carries `healthy` + `last_failure`. A failed step is RECORDED on the train (the catalog shows "broken at step N" without re-running). Loop: replay → break → SEE it (the `blackbox` path) → repair (`playbook_edit_step`/objects) → replay; a clean run sets `recovered:true`.

## Objects — the Pixel Match book (Pro)
Teach a button by its pixel fingerprint and re-find it anywhere — works where UIA falls short (games, custom UIs, WebView).
- `learn_button({label, x, y, w, h, app?})` — fingerprint a bbox (errors on a uniform region — include the glyph).
- `find_button({label})` → `{found, x, y, rect, match_pct, candidates}` or `{found:false}`.
- `click_button({label})` — find + click; the visual action recorded by trains. On replay it resolves the object by its EXACT recorded name (a renamed/deleted object fails the step instead of clicking a lookalike).
- `track_object({label})` → `{found, moved, dx, dy, searched}`. `object_thumbnail({label})` → 48px sample. `list_buttons()` · `delete_button({label})`.

## Linked objects (Pro)
Deterministic analysis of `playbooks.json` + `buttons.json` (no screenshot/model).
- `object_links()` → `{objects:[{label, registered?, app, trains, last_seen_rect}], orphan_refs, unused}`.
- `objects_used_by_train({task})` → objects a train links to. `trains_using_object({label})` → trains linking an object (move/relearn one → see the N affected).

## Run commands on the machine (two distinct tools)
- `open_run({command})` — opens the OS Run dialog (Win+R), types, Enter. **Visible, OS-mediated.** For GUI apps (`mspaint`), terminals (`cmd`/`powershell`/`wt`), or a `cmd /c …` one-liner that closes itself. After opening a terminal, drive it: `type_text` + `press_key({key:"Enter"})`.
- `run_command({command, cwd?, detach?, timeout_ms?})` — runs **locally** via `cmd /C`, captures `{stdout, stderr, exit_code, timed_out, duration_ms}`. `detach:true` launches a GUI/long-running process and returns at once without capturing. **OFF by default** (Settings → Advanced). For trains it's classified `irreversible` — a command can't be undone.
