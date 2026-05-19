# verdesk-skill

Claude Code skill para usar **[Verdesk](https://github.com/chamilonster/verdesk)** — un servidor MCP local de control de escritorio optimizado para modelos de lenguaje.

Verdesk expone tu pantalla y el control del PC vía MCP, pero a diferencia de un screenshot tool genérico **manda solo deltas + texto plano**, no la pantalla completa cada turno. Resultado: menos tokens, más velocidad, mejor decisión de la IA.

## Instalación

```bash
# Dentro de Claude Code:
/plugin marketplace add https://github.com/chamilonster/verdesk-skill
/plugin install verdesk@verdesk-skill
```

Después, registrá el endpoint MCP que viene de tu instalación de Verdesk:

```bash
claude mcp add verdesk http://127.0.0.1:47802/mcp
```

Si Verdesk está corriendo en otro PC, el endpoint cambia. Settings → 04 ACCESO en la UI de Verdesk te arma el comando exacto y un archivo de invitación para pasar por pendrive/mail.

## Qué hace esta skill

Le da contexto a la IA cliente (Claude Code u otro consumidor MCP) sobre **cuándo usar Verdesk** y **qué tool elegir** para cada tarea visual. Sin esta skill el modelo igual puede consumir las tools, pero pierde el patrón "look → decide → action → verify" que minimiza tokens.

## Qué hace Verdesk (resumen)

- **Captura modulada por celdas** (grilla 12×8 sobre el viewport). Solo te devuelve las celdas que cambiaron desde la última captura.
- **Lectura de texto plano** de zonas de pantalla — la IA recibe el texto listo, sin imágenes que procesar.
- **Profiles de modulación** persistentes — recetas (`resolución × calidad × modo de color × ajustes`) atadas a targets (monitor, ventana, app). La IA puede listar/cargar/guardar/ratear.
- **Acciones**: click semántico vía UI Automation, click sobre texto, click sobre coordenadas, escritura, scroll, comandos de shell.
- **Memoria visual cross-session** — historial buscable por hash perceptual.

## Modos de acceso

Verdesk tiene 3 modos seleccionables desde Settings → 04 ACCESO:

| Modo | Cómo conecta | Auth |
|------|--------------|------|
| **Local** | `127.0.0.1` — mismo PC | Sin auth |
| **LAN** | IP del host en la red local | Handshake con popup en el server la primera vez |
| **WAN** | Tunel SSH hasta el host | SSH + handshake con popup la primera vez |

Cada modo genera un prompt copy-paste listo para pegar en el cliente. WAN/LAN además generan un archivo `<hostname>_verdesk.md` con todo lo necesario (endpoint, comando SSH, link a esta skill).

## Repo principal de Verdesk

→ [github.com/chamilonster/verdesk](https://github.com/chamilonster/verdesk)

## Licencia

MIT.
