# Conectar el MCP de Apify

El [servidor MCP de Apify](https://docs.apify.com/platform/integrations/mcp) le permite al
modelo invocar Actors directamente. En la Fase 0 es todo el "backend" que hay.

Hay dos formas. **La hospedada es la recomendada** para la Fase 0: no deja el token en un
archivo local.

---

## Opción A — Servidor hospedado (recomendada)

Apuntar el cliente a `https://mcp.apify.com`, que usa Streamable HTTP con OAuth. La primera
conexión abre el navegador para autorizar; no hay token que copiar ni archivo que proteger.

Se puede filtrar por query string:

```
https://mcp.apify.com?tools=<ACTOR-DE-X>
```

En Claude Code:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=<ACTOR-DE-X>"
```

## Opción B — Servidor local vía npx

Requiere Node.js 18+. Usar [`apify.json`](apify.json) como plantilla.

En **Claude Desktop**, pegar el contenido en el archivo de configuración:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Después **cerrar Claude Desktop por completo** (no minimizar) y volver a abrirlo. El ícono 🔌
confirma la conexión.

En **Claude Code**:

```bash
claude mcp add apify --env APIFY_TOKEN=$APIFY_TOKEN \
  -- npx -y @apify/actors-mcp-server --tools <ACTOR-DE-X>
```

---

## Por qué el filtro `--tools` no es opcional

Sin él, el servidor expone el catálogo completo de Actors de Apify. Eso infla la metadata de
herramientas que el modelo tiene que leer en cada turno, gasta contexto y hace que el ruteo se
confunda entre actores parecidos. **Se expone un solo actor: el elegido en
[`../docs/apify-actor-x.md`](../docs/apify-actor-x.md).**

## Verificación

Pedirle al modelo que liste las herramientas disponibles. Tiene que aparecer el actor elegido,
y solo ese. Si aparecen decenas de Actors, el filtro no se aplicó.

## Nota sobre Claude Desktop vs Claude Code

El documento de visión original asume Claude Desktop. **Para la Fase 0 conviene Claude Code**:
el MCP funciona igual, pero además el JSON crudo de cada corrida queda escrito directamente en
`datos/crudo/` sin pasos manuales de copiar y pegar — que es exactamente lo que la Fase 1
necesita como fixtures de sus tests.

Claude Desktop sigue siendo válido si preferís la interfaz de chat; en ese caso los JSON hay
que guardarlos a mano.

> El documento de visión propone `npx -y @apify/cli mcp run`. Ese no es el servidor MCP: el
> paquete correcto es `@apify/actors-mcp-server`.
