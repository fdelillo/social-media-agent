# Conectar el MCP de Apify

El [servidor MCP de Apify](https://docs.apify.com/platform/integrations/mcp) le permite al
modelo invocar Actors directamente. En la Fase 0 es todo el "backend" que hay.

Hay tres formas. **La primera es la recomendada**: no deja el token en ningún archivo y sirve
igual en Claude Code, Claude Desktop y la web.

---

## Opción A — Conector de la cuenta de claude.ai (recomendada)

Agregar Apify como conector desde la configuración de claude.ai. Usa OAuth: se autoriza una vez
contra tu cuenta de Apify y queda disponible en todos los clientes y sesiones, sin token que
copiar ni archivo que proteger.

Aparece en `claude mcp list` como `claude.ai Apify`.

## Opción B — Servidor hospedado, configurado por proyecto

Lo mismo, pero declarado solo en este repo. Sirve si querés **filtrar los actores expuestos**
(ver abajo), algo que el conector de cuenta no permite:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=<ACTOR-DE-X>"
```

Requiere autenticar aparte con `/mcp`. **No conviene tener las dos a la vez**: quedan dos
entradas de Apify compitiendo, y la que no se autenticó falla en silencio.

## Opción C — Servidor local vía npx

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

## Sobre el filtro de actores

Sin el parámetro `tools`, el servidor **no** expone el catálogo entero de Apify como
herramientas: expone un conjunto chico de herramientas de *descubrimiento* — buscar actores en
la store, consultar el detalle de uno, invocarlo — más las de documentación. Es un modelo
dinámico, y es perfectamente usable.

Para la Fase 0 esa configuración por defecto es incluso **preferible**, porque todavía estamos
eligiendo el actor y las herramientas de búsqueda sirven justo para eso.

El parámetro `tools` sirve para lo contrario: fijar uno o más actores como herramientas
dedicadas, una vez elegidos. Vale la pena cuando el actor ya está decidido y se quiere evitar
que el modelo salga a explorar la store en cada corrida.

## Verificación

Correr `claude mcp list`. Apify tiene que figurar como `✔ Connected`. Si figura
`! Needs authentication`, falta pasar por `/mcp` y autorizar.

Las herramientas MCP se cargan al iniciar la sesión: si el servidor se conectó recién, hay que
reiniciar Claude Code para poder usarlas.

## Nota sobre Claude Desktop vs Claude Code

El documento de visión original asume Claude Desktop. **Para la Fase 0 conviene Claude Code**:
el MCP funciona igual, pero además el JSON crudo de cada corrida queda escrito directamente en
`datos/crudo/` sin pasos manuales de copiar y pegar — que es exactamente lo que la Fase 1
necesita como fixtures de sus tests.

Claude Desktop sigue siendo válido si preferís la interfaz de chat; en ese caso los JSON hay
que guardarlos a mano.

> El documento de visión propone `npx -y @apify/cli mcp run`. Ese no es el servidor MCP: el
> paquete correcto es `@apify/actors-mcp-server`.
