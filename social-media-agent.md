# 📋 Documento de Visión de Proyecto: Agente de Social Listening & Sentiment Analysis (MVP)

Este documento define la arquitectura y el marco de trabajo para construir un agente autónomo de IA capaz de rastrear menciones de personas o marcas en X (Twitter) e Instagram, analizar el sentimiento de los textos y generar informes analíticos ejecutivos.

---

## 🎯 1. Alcance del MVP (Mínimo Viable)
*   **Entrada del usuario:** El nombre de una persona o marca comercial (ej. "Elon Musk", "Nike").
*   **Fuentes de datos:** Extracción de textos/copys de posteos recientes en X (Twitter) e Instagram usando los actores de Apify.
*   **Procesamiento:** Clasificación de sentimiento en tiempo real (Positivo, Neutro, Negativo) mediante modelos LLM comerciales (Claude Pro / ChatGPT Plus).
*   **Salida:** Un reporte estructurado en formato Markdown.

---

## 🔌 2. Configuración de la Cuenta de Datos (Apify)
Para alimentar al agente sin programar infraestructura propia en esta fase, utilizaremos el plan gratuito de Apify (\$5 USD/mes de crédito).

1. Registrate en [apify.com](https://apify.com) (no requiere tarjeta de crédito).
2. Ve a **Settings** ➔ **Integrations** y copia tu **API Token** (lo necesitarás para ambas integraciones).

---

## 🧠 3. Opción de Conexión A: Model Context Protocol (MCP)
*Ideal para tu uso personal en Claude Desktop (Pro).*

El protocolo MCP permite a la aplicación de escritorio de Claude comunicarse directamente con la infraestructura de Apify para ejecutar scrapers en tiempo real utilizando herramientas nativas.

### Pasos para la configuración:
1. Abre los ajustes de tu **Claude Desktop**.
2. Ve a la sección de configuración de desarrollador o abre directamente el archivo de configuración en tu sistema:
   - **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
   - **Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
3. Pega la siguiente estructura en el archivo JSON (reemplazando el token por el tuyo):

```json
{
  "mcpServers": {
    "apify": {
      "command": "npx",
      "args": [
        "-y",
        "@apify/cli",
        "mcp",
        "run"
      ],
      "env": {
        "APIFY_TOKEN": "TU_API_TOKEN_DE_APIFY"
      }
    }
  }
}
```
4. Reinicia Claude Desktop. Verás un ícono de un enchufe que indica que Claude ahora puede invocar los "Actores" de tu cuenta de Apify de forma autónoma.

---

## 🔌 4. Opción de Conexión B: Actions (OpenAPI / Swagger)
*Ideal para crear un Custom GPT en ChatGPT Plus y compartirlo con otros usuarios.*

Para que ChatGPT sepa cómo pedirle datos a Apify, debes ir a la configuración de tu **Custom GPT ➔ Actions ➔ Create new action** y pegar el siguiente esquema OpenAPI simplificado:

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "Apify Social Listening API",
    "description": "API para ejecutar tareas de scraping en X e Instagram utilizando Apify.",
    "version": "1.0.0"
  },
  "servers": [
    {
      "url": "https://apify.com"
    }
  ],
  "paths": {
    "/actor-tasks/{taskId}/runs": {
      "post": {
        "summary": "Ejecutar una tarea de scraping preconfigurada",
        "operationId": "runActorTask",
        "parameters": [
          {
            "name": "taskId",
            "in": "path",
            "required": true,
            "description": "El ID de la tarea configurada en Apify (ej. tu scraper de X o IG)",
            "schema": {
              "type": "string"
            }
          }
        ],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "searchQueries": {
                    "type": "array",
                    "items": {
                      "type": "string"
                    },
                    "description": "Palabra clave o nombre a buscar en la red social."
                  }
                }
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Tarea iniciada con éxito."
          }
        }
      }
    }
  }
}
```
*Nota de autenticación en ChatGPT:* En la configuración de la Acción, selecciona **Authentication: API Key**, elige el tipo **Bearer** y pega tu API Token de Apify.

---

## 🤖 5. Especificación del Agente (El Prompt Maestro)

```text
Eres un Agente Experto en Social Listening, Analítica de Datos y Reputación Digital.

Tu único objetivo es recibir el nombre de una persona o marca comercial proporcionado por el usuario, analizar el contexto de sus menciones en redes sociales (Instagram y X), y estructurar un reporte analítico de sentimiento.

### INSTRUCCIONES OPERATIVAS:
1. Solicita al usuario el nombre de la marca o persona a analizar.
2. Utiliza la acción API/MCP configurada para extraer las menciones recientes en X e Instagram basadas en ese nombre.
3. Lee minuciosamente cada posteo recibido. Evalúa el tono, uso de ironía, adjetivos y emojis para clasificar de forma precisa el sentimiento en: Positivo (🟢), Neutro (🟡) o Negativo (🔴).

### FORMATO DE SALIDA REQUERIDO (STRICT MARKDOWN):
Genera siempre tu respuesta utilizando exactamente la siguiente estructura:

# 📊 Reporte de Escucha Social: [Insertar Nombre]
## 📈 Resumen Ejecutivo
- **Total de menciones analizadas:** [X]
- **Distribución de Sentimiento:** [🟢 X% Positivo | 🟡 X% Neutro | 🔴 X% Negativo]

## 📱 Desglose por Red Social

### 🐦 X (Twitter)
- **Análisis de Narrativa:** [Resumen analítico de 2 oraciones sobre qué se está diciendo principalmente].
- **Menciones Clave:**
  - "[Texto del post]" - *Sentimiento: [Positivo/Negativo/Neutro]*

### 📸 Instagram
- **Análisis de Narrativa:** [Resumen analítico de 2 oraciones sobre la tendencia de los comentarios].
- **Menciones Clave:**
  - "[Texto del copy]" - *Sentimiento: [Positivo/Negativo/Neutro]*

## 💡 Conclusiones y Recomendaciones Estratégicas
- [Brinda una conclusión cualitativa sobre la percepción actual de la marca/persona].
- [Sugiere 2 acciones concretas que la marca/persona debería tomar basándose en los datos].
```

---

## 🚀 6. Hoja de Ruta de Escalabilidad (Fase Post-Apify)
Para reemplazar la dependencia externa en el futuro y lograr un entorno 100% gratuito y escalable:
1. **Módulo de Extracción propio:** Microservicios en Go o Python usando librerías open-source como `Playwright`.
2. **Orquestación en n8n:** Configurar flujos de tareas programadas (Cron) instalados en tu contenedor de Docker local.
3. **Procesamiento por API:** Enviar los datos directo a las APIs oficiales de OpenAI/Anthropic por lote, pagando solo centavos por token.
