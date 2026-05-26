# verdesk-skill

Claude Code skill para usar **[Verdesk](https://github.com/chamilonster/verdesk)** — un servidor MCP local de control de escritorio optimizado para modelos de lenguaje.

Verdesk expone tu pantalla y el control del PC vía MCP, pero a diferencia de un screenshot tool genérico **manda solo deltas + texto plano**, no la pantalla completa cada turno. Resultado: menos tokens, más velocidad, mejor decisión de la IA.

## Cómo se usa

El user copia un prompt de Settings → 04 ACCESO en la UI de Verdesk y lo pega en su cliente Claude Code. El prompt contiene el link a este `SKILL.md` y los datos de conexión.

La IA cliente lee el `SKILL.md`, lo absorbe como skill local (`~/.claude/skills/verdesk/SKILL.md`), y ejecuta el bootstrap automático: genera keypair SSH, postea la pub al server (un popup le aparece al user para aprobar una sola vez), abre el tunnel SSH, y registra el server MCP en `.mcp.json`. Después le pide al user que reinicie el cliente para que recargue los servers.

Detalle completo del bootstrap autónomo en [`SKILL.md`](./SKILL.md#bootstrap-autónomo-primera-vez-en-este-cliente).

### Modo Local (mismo PC)

Cuando Verdesk corre en la misma máquina que el cliente, el bootstrap se reduce a:

```bash
claude mcp add --transport http verdesk http://127.0.0.1:47802/mcp
```

(El `47802` es el `mcp_port` default, configurable en Settings.)

## Qué hace esta skill

Le da contexto a la IA cliente (Claude Code u otro consumidor MCP) sobre **cuándo usar Verdesk** y **qué tool elegir** para cada tarea visual. Además contiene el manual de bootstrap autónomo para conexiones remotas (SSH tunnel + autorización de pub key). Sin esta skill el modelo no sabría cómo conectarse a una instalación remota de Verdesk.

## Qué hace Verdesk (resumen)

- **Captura modulada por celdas** (grilla 12×8 sobre el viewport). Solo te devuelve las celdas que cambiaron desde la última captura.
- **Lectura de texto plano** de zonas de pantalla — la IA recibe el texto listo, sin imágenes que procesar.
- **Profiles de modulación** persistentes — recetas (`resolución × calidad × modo de color × ajustes`) atadas a targets (monitor, ventana, app). La IA puede listar/cargar/guardar/ratear.
- **Acciones**: click semántico vía UI Automation, click sobre texto, click sobre coordenadas, escritura, scroll, comandos de shell.
- **Memoria visual cross-session** — historial buscable por hash perceptual.

## Modos de acceso

Verdesk tiene 2 modos seleccionables desde Settings → 04 ACCESO:

| Modo | Cómo conecta | Auth |
|------|--------------|------|
| **Local** | `127.0.0.1` — mismo PC | Sin auth |
| **Remote** | Tunnel SSH al host (Tailscale o IP pública + UPnP) | SSH pub key + 1 popup de aprobación al inicio |

El modo Remote unifica los viejos LAN y WAN: todo cliente externo entra por SSH tunnel cifrado. Una sola pub key autorizada vale para siempre — sin tokens MCP recurrentes ni popups repetidos.

## Repo principal de Verdesk

→ [github.com/chamilonster/verdesk](https://github.com/chamilonster/verdesk)

## Licencia

MIT.
