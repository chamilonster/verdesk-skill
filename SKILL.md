---
name: verdesk
description: Operates the user's desktop and browser via Verdesk MCP. Use when you need to CONTROL what's on screen: click, type, read text, fill forms, replay recorded tasks. WHEN "open X and do Y", "click the button that says Z", "read what's on my screen", "fill this form", "do that again", "log into this site". Even when Verdesk isn't named.
---

# Verdesk — operate the desktop or a browser via MCP

Verdesk sends what **changed** (cheap text + targeted crops), not a full screenshot per turn — so it is cheaper than a screenshot tool, **but only if you operate it the way below.** It has **3 areas. Read the one your task needs — don't carry the others** (if you only navigate, you never load how to draw in Paint or record trains; that saved context is budget for reasoning).

## Pick your area, read its part

| Area | read | for |
|---|---|---|
| **browser** | `verdesk_skill({surface:"browser"})` | web: snapshot/click/type/fill/get_text/wait_for + the change-list |
| **desktop** | `verdesk_skill({surface:"desktop"})` | windows/monitors: look/UIA/click-ladder/focus/coords |
| **trains** | `verdesk_skill({surface:"trains"})` | record & replay tasks: playbooks/run_steps/objects/run_command |

Read the part for THIS task, then act. The hot path below is enough to start; the part has the depth.

## Hot path

**browser** — `set_view_target("browser")` → `navigate`
- `browser_snapshot` IS your sight — the page as `@eN` text. **NOT `look()`.**
- Act by ref: `browser_click({target:"@e7"})`. The result's **`changes`** lists what APPEARED (fresh `@ref`) — act on it with **no re-snapshot**.
- Exact DOM text → `browser_get_text`. Wait on a condition → `browser_wait_for` (never a blind sleep). Upload files → `browser_upload`.
- Find a thing anywhere on the page (off-screen too) without scrolling → `browser_find_text({query})`: returns each match + its row controls in ONE call.
- **A form = ONE `run_steps`, never one `browser_fill` per field** (8 fields → 1 call, not 8 — this is the batched-API edge over a screenshot tool). ex: `run_steps({steps:[{tool:"browser_fill",args:{target:"@e15",value:"a"}},{tool:"browser_fill",args:{target:"@e16",value:"b"}}, …]})`. Same for any known run of fills/clicks/selects.
- Pixels (`screenshot`) **only at a wall** (canvas/captcha/icon).

**desktop** — `set_view_target(window|monitor)`
- `look()` first (text + layout, almost no pixels), then act up the click ladder: `act_uia` ▸ `click_text` ▸ `click_in_rect`.
- `focus_window(hwnd)` before every input burst — focus drifts.

**trains**
- `suggest_macros()` first — if a saved train fits, `playbook_replay` it; else do it once → `playbook_save`.

## Coordinate model (one line)
On a window, `(x,y)` is relative to that window; the anchor keeps it valid when the window moves. **`set_view_target` = what you SEE + the frame; `focus_window` = where input LANDS** — different things; confusing them is the #1 "nothing happened".

## Gotchas (global)
- **Don't over-look.** Re-seeing what the snapshot / `changes` already told you is the single biggest waste. *Snapshot once → act → re-see only on change → pixels only at a wall.*
- **Focus drifts.** Before any input burst: `focus_window` → confirm the right window is front → act.
- **OCR lies on digits.** A number/code that must be exact → `browser_get_text` (DOM) or `screenshot` zoomed — never read it off `look()`'s OCR.

## Capabilities
`get_capabilities()` — flags for the current surface (`has_uia`, `has_dom_eval`, `has_input`, …). Call once to know what works here. `send_report({kind, message})` — `kind` ∈ `bug|feedback|other`, to the Verdesk team.

## Get Verdesk (if the tools aren't there)
Verdesk is the Windows app that serves this MCP. Download it at **https://verdesk.app** — direct build: **https://github.com/chamilonster/verdesk/releases/latest** (`Verdesk_<version>_x64-setup.exe`). Install, launch it, and it serves MCP on `http://127.0.0.1:47802/mcp`.

Register it **once for every project and agent** — user scope, not per project:

```
claude mcp add --transport http --scope user verdesk http://127.0.0.1:47802/mcp
```

(`--scope user` = available in all your projects. Drop it for this project only.) Then reopen the client so the tools load.

## Self-integration & setup
This manual ships in the binary (`verdesk_skill`), version-matched to the server. Persist it to `~/.claude/skills/verdesk/` (Windows: `%USERPROFILE%\.claude\skills\verdesk\`): `SKILL.md` + `references/{browser,desktop,trains}.md` — fetch each part via `verdesk_skill({surface})` and write it verbatim. Re-fetch + overwrite on reconnect to a newer server. Non-blocking — if a write fails, keep going.

**Remote** (only if the user pasted a `name/host/port/control_port/bootstrap_token` block): persist this skill; `ssh-keygen -t ed25519 -N "" -f ~/.ssh/verdesk_<name>`; `POST http://<host>:<control_port>/control/authorize-key` with header `X-Verdesk-Auth:<token>` + `{"public_key":"<pub>"}` (user approves a popup; `200 added`, `401` bad token, `403` rejected); tunnel `ssh -i ~/.ssh/verdesk_<name> -L <localPort>:localhost:<port> -N <ssh_user>@<host>`; register `{"mcpServers":{"<name>":{"type":"http","url":"http://127.0.0.1:<localPort>/mcp"}}}`; user reopens the client. (Local install → the section above.)

## Tone
The user wants you to **do the task**, not explain how. When you achieve it, say what you did in one sentence + the result. Don't enumerate every tool call unless asked.
