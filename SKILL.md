---
name: verdesk
description: Control de escritorio remoto vía MCP. El usuario te pidió conectarte a un Verdesk (suyo o de otro) y operarlo. Esta skill cubre conexión + operación. Optimizada para tokens — la IA consume deltas + texto plano, no screenshots completos.
---

# Verdesk

El usuario te pidió conectarte a un escritorio remoto (o local) que corre Verdesk y operarlo en su nombre. Esta skill enseña dos cosas, en orden:

1. **Cómo conectarte** al Verdesk (bootstrap por variante).
2. **Cómo operarlo** con las tools del MCP.

Lee del prompt del usuario los datos: `variant`, `host`, `port`, `name`, `auth`. Si falta alguno, pídeselos antes de seguir.

---

## 1. Conexión

Decide el flow por el valor de `variant`.

### variant=local | lan

```
claude mcp add --transport http <name> http://<host>:<port>/mcp \
  --header "X-Verdesk-Auth: <auth>" \
  --header "X-Verdesk-Client-Name: $COMPUTERNAME"
```

(En cmd.exe usar `%COMPUTERNAME%`, en bash/zsh `$HOSTNAME`.)

Verifica con `claude mcp list` que `<name>` aparezca. Reporta `✓ Conectado a Verdesk (modo <variant>)`.

### variant=ssh-directo

1. Pide al usuario el `<user>` SSH del host destino si no lo dio.
2. Túnel en background: `ssh -L <port>:localhost:<port> -N -f <user>@<host>` (o sin `-f` y manejarlo como proceso background del shell).
3. Espera 2s para que el bind se estabilice.
4. Igual que en local/lan, pero contra `http://127.0.0.1:<port>/mcp` (no contra `<host>`).
5. ✓ Reporta conectado.

### variant=tailscale

El host es una IP `100.x.x.x` que **solo es alcanzable desde dentro de la overlay Tailscale**. Si fallas el preflight, el `ssh -L` da timeout sin mensaje útil. Sigue los 4 pasos en orden.

#### a) ¿Tailscale instalado?

- Windows: `where tailscale.exe`
- macOS/Linux: `which tailscale`

Si retorna path → sigue a (b).

Si no retorna nada, ofrece auto-instalar:

> "Necesitas Tailscale instalado para alcanzar `<host>`. ¿Lo bajo e instalo? Requiere tu permiso (UAC en Windows / sudo en Linux/macOS)."

Si el usuario acepta:

- **Windows** (PowerShell): `Invoke-WebRequest -Uri https://pkgs.tailscale.com/stable/tailscale-setup-latest.exe -OutFile "$env:TEMP\ts-setup.exe"; Start-Process -FilePath "$env:TEMP\ts-setup.exe" -Verb RunAs -Wait`
- **macOS**: `brew install --cask tailscale` (asume Homebrew).
- **Linux**: `curl -fsSL https://tailscale.com/install.sh | sh`

Si el usuario rechaza: para, di *"No puedo continuar. Verdesk está detrás de CGNAT y solo es accesible vía Tailscale."*

#### b) ¿Tailscale logueado?

```
tailscale status --json
```

Parsea `BackendState`:

- `Running` → sigue a (c).
- `NeedsLogin` / `Stopped` / `NoState` → *"Abre Tailscale desde el tray del SO y haz login con tu cuenta. Avísame cuando termines."* — espera respuesta del usuario y reintenta desde (b).
- `NeedsMachineAuth` → *"Tu dispositivo está pendiente de aprobación en el admin panel. Entra a https://login.tailscale.com/admin/machines y apruébalo. Avísame cuando termines."*

#### c) ¿El peer `<host>` es visible en mi tailnet?

Polling cada 3s, máximo 60s:

```
tailscale status --json
```

Busca en `.Peer[*].TailscaleIPs[]` que aparezca `<host>`.

- Aparece → sigue a (d).
- No aparece tras 60s → *"El peer `<host>` no está en tu tailnet. Pídele al dueño del Verdesk que te comparta el node vía Tailscale Sharing: https://tailscale.com/kb/1084/sharing — que genere un share link, lo ábres en tu browser y aceptas. Avísame cuando termines."*

#### d) SSH túnel + MCP add

1. `ssh -L <port>:localhost:<port> -N -f <user>@<host>` (pide `<user>` si no estaba).
2. Espera 2s.
3. `claude mcp add --transport http <name> http://127.0.0.1:<port>/mcp --header "X-Verdesk-Auth: <auth>" --header "X-Verdesk-Client-Name: $COMPUTERNAME"`
4. ✓ Reporta: *"Conectado a Verdesk vía Tailscale (peer `<host>`)."*

---

## 2. Operación (tools MCP)

Una vez conectado, consumes las tools del MCP. Reglas duras:

1. **Empieza con `look()`** — primitiva barata, devuelve texto plano + layout, cero pixels por default.
2. **Refina con zona**, no recapturando todo: `look(zone=...)` o `refine_cell(...)`.
3. **Acción**: usa la primitiva más alta. UIA (`act_uia`) > texto (`click_text`) > coords (`click_at`).
4. **Texto en pantalla**: léelo del campo `text` que devuelve `look()`. Ya viene plano.
5. **Esta skill es para escritorio** (Excel, Word, IDEs, terminales, juegos, apps custom). No la uses para scraping de páginas web tradicional.

### Ver pantalla

| Tool | Cuándo |
|---|---|
| `look(zone?, want?, mode?)` | Default. `mode`: `glance` (barato) \| `detail`. `want`: `["text","layout"]` (default) \| suma `"visual"` si necesitas pixels. |
| `refine_cell(cell_id, quality?)` | Re-capturar UNA celda de la grilla 12×8 con calidad superior. |
| `get_buffer_state(include_thumbnails?)` | Metadata del buffer activo. |

### Acción

| Tool | Cuándo |
|---|---|
| `list_uia_elements(visible_only?, max_depth?)` | Inventario del árbol UIA — `auto_NNN` ids, name, control_type, patterns. |
| `act_uia(id, action)` | Acción semántica. `action`: `{kind: invoke\|set_value\|toggle\|select\|expand\|collapse, value?}`. |
| `click_text(query, occurrence?)` | Click sobre substring. `occurrence`: `first` (default) \| `last` \| `nth` \| `all`. |
| `click_collage(id)` | Click sobre el centro de un collage del último `look()`. |
| `click_at(x, y)` | Click bajo nivel por coords. Último recurso. |
| `send_keys(text)` · `press_key(key, ctrl?, alt?, shift?, win?)` | Tipear texto, tecla nombrada o combo. |
| `scroll(direction, amount_px)` | `up` \| `down` \| `left` \| `right`. |
| `focus_window(hwnd_hex)` | Traer ventana al frente antes de mandar input. |

### Memoria visual entre turnos

| Tool | Cuándo |
|---|---|
| `list_history(reason?, url_contains?, limit?)` | Snapshots archivados. |
| `query_history(phash, threshold?)` | "¿Vi esto antes?" Memoria asociativa cross-session. |

### Profiles de modulación

| Tool | Cuándo |
|---|---|
| `list_profiles(target_match?, min_rating?, creator_kind?)` | Recetas guardadas. |
| `load_profile(id? \| target_match?)` | El mejor profile para un target (ej. `target_match="monitor:primary"`). |
| `save_profile(target_type, target_match, params, creator)` | Guardar una receta. |
| `rate_profile(id, rating?)` | Reportar performance (0.0–1.0). |

### Shell remoto

| Tool | Cuándo |
|---|---|
| `run_command(command, cwd?, timeout_ms?)` | Comando en el PC remoto. Útil para evitar 20 clicks. Devuelve stdout/stderr/exit_code. |

---

## Tono

El usuario quiere que **hagas la tarea**, no que narres cada tool call. Al terminar: una frase con qué hiciste + resultado. Sin desglose salvo que lo pida.
