# verdesk-skill

Skill para Claude Code (y otros clientes MCP) que enseña a la IA a usar **[Verdesk](https://verdesk.app)** — un servidor MCP para que vos veas la pantalla del user y operes su PC, optimizado para gastar pocos tokens.

## Cómo se usa

El user copia un prompt desde Settings → 04 ACCESO en la UI de Verdesk y lo pega en su cliente. El prompt incluye el link a este `SKILL.md` + los datos de conexión.

La IA lee el manual, se lo absorbe como skill local, y se conecta sola. El user solo aprueba **una vez** un popup que aparece en su pantalla.

Detalle de los pasos: [`SKILL.md`](./SKILL.md#conexión-inicial-primera-vez).

### Modo Local (mismo PC)

Cuando Verdesk corre en la misma máquina que el cliente, no hay bootstrap. El user solo registra el server:

```bash
claude mcp add --transport http verdesk http://127.0.0.1:47802/mcp
```

## Qué le da Verdesk a la IA

- **Ver pantalla**: capturas eficientes que gastan menos tokens que un screenshot tool genérico.
- **Leer texto plano** de zonas de pantalla — la IA recibe el texto listo, sin imágenes que procesar.
- **Tomar acciones**: click semántico, click por texto visible, click por coords, escritura, scroll, comandos de shell.
- **Memoria visual** entre turnos.

## Modos de acceso

| Modo | Cuándo |
|------|--------|
| **Local** | Cliente IA y Verdesk en el mismo PC. |
| **Remote** | Cliente IA en otra máquina. Conexión cifrada con autorización por clave del lado del user. |

## Licencia

MIT.
