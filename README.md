# social-media-agent

Agente de reputación digital: recibe el nombre de una marca o persona, trae menciones
recientes de X, clasifica el sentimiento de cada una y emite un informe ejecutivo en Markdown.

**Sin infraestructura.** No hay Docker, ni n8n, ni Postgres, ni servicios que levantar. Los
datos salen de [Apify](https://apify.com) y el análisis lo hace un LLM. La meta es que alguien
lo instale con un comando y lo corra.

> **Estado actual: Fase 0 (validación).** Todavía no hay código ejecutable. Lo que hay son los
> prompts, la configuración del MCP y el andamiaje para medir si el enfoque funciona antes de
> escribir el CLI. Ver [`docs/fase-0.md`](docs/fase-0.md).

## Qué NO es esto

Este repo tiene un hermano, [`../agente-clipping`](../agente-clipping), que cubre el mismo
dominio con el enfoque opuesto: Docker Compose, n8n, FastAPI, Postgres y Groq, con el objetivo
declarado de aprender infraestructura. **La frontera entre los dos es deliberada.** Quedan del
otro lado, y no se construyen acá:

- Persistencia, deduplicación entre corridas, histórico y evolución en el tiempo.
- Orquestación y tareas programadas.
- Todo el pipeline de video: descarga, transcripción con Whisper, clipping con FFmpeg.

Acá cada corrida es autocontenida: entra un nombre, sale un informe.

Lo único que ambos comparten a propósito es el **esquema normalizado de una mención** (fuente,
id externo, autor, texto, url, fecha, métricas), para que los datos sean intercambiables.

## Alcance

| | |
| :--- | :--- |
| **Fuente** | X (Twitter), vía un Actor de Apify. Instagram queda para después, detrás de la misma normalización. |
| **Análisis** | Sentimiento por mención: 🟢 positivo · 🟡 neutro · 🔴 negativo, con score y justificación. |
| **Salida** | Un informe Markdown: resumen ejecutivo, narrativa, menciones clave, conclusiones. |

El formato exacto del informe está en [`social-media-agent.md`](social-media-agent.md), el
documento de visión que originó el proyecto.

## Cómo se usa hoy (Fase 0)

1. Crear una cuenta en [apify.com](https://apify.com) y copiar el API token desde
   **Settings → Integrations**.
2. `cp .env.example .env` y completar `APIFY_TOKEN`.
3. Conectar el MCP de Apify siguiendo [`mcp/README.md`](mcp/README.md).
4. Seguir los pasos de [`docs/fase-0.md`](docs/fase-0.md).

En la Fase 0 el "agente" es Claude Code (o Claude Desktop) leyendo los prompts de
[`prompts/`](prompts/). No hace falta API key de Anthropic: el modelo es el de tu suscripción.

## Cómo se va a usar (Fase 1)

```bash
pipx install git+https://github.com/fdelillo/social-media-agent
sma analizar "Nike" --limite 100
```

## Estructura

```
docs/          decisiones de diseño, guía de la Fase 0, notas del actor de Apify
prompts/       los dos prompts: uno clasifica, otro redacta
mcp/           configuración del servidor MCP de Apify
datos/crudo/   JSON tal cual sale de Apify (no versionado)
datos/dorado/  menciones etiquetadas a mano — la vara contra la que se mide el prompt
reportes/      informes generados (se versionan solo los ejemplos)
```

## Costos

Los dos únicos costos son Apify y el LLM, y **el que aprieta es Apify**:

| | Costo aproximado |
| :--- | :--- |
| Apify | ~$5/mes de crédito en el plan gratuito · los actores de X rondan $0.15–$0.40 por 1.000 tweets, más un *start fee* por corrida |
| LLM (Fase 1, por corrida de 100 menciones) | ~$0.35 |

> ⚠️ **El crédito gratuito puede no alcanzar para usar este proyecto.** Varios de los actores
> de X más populares restringen a los usuarios del plan gratuito, y al menos uno les prohíbe el
> acceso por API — que es justamente por donde entran el MCP y el CLI. Cuáles funcionan en
> gratuito es una pregunta abierta que se responde probando; el estado del relevamiento está en
> [`docs/apify-actor-x.md`](docs/apify-actor-x.md).

De ahí sale una regla de diseño: **cada corrida guarda su JSON crudo en `datos/crudo/`**, y
nada se vuelve a scrapear para reprocesarlo. Reanalizar es gratis; volver a scrapear no.
