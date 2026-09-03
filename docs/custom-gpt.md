# Anexo: Custom GPT (diferido)

El documento de visión propone un Custom GPT como vía de distribución. **No está en el alcance
actual**: el requisito de "producto compartible" se resuelve con `pipx install` del CLI, que no
ata el proyecto a otro proveedor.

Este anexo queda por si en algún momento querés específicamente distribuirlo dentro de ChatGPT.
El esquema de abajo es el del documento original, **corregido**.

---

## Qué estaba mal en el esquema original

**El servidor apuntaba al sitio, no a la API.** Decía `"url": "https://apify.com"`; la API vive
en `https://api.apify.com/v2`. Con el host equivocado, todas las llamadas fallan.

**El endpoint elegido era asíncrono.** `POST /actor-tasks/{taskId}/runs` arranca la corrida y
devuelve un objeto *run* con estado `RUNNING`, sin datos. Para obtener las menciones hay que
hacer polling del dataset hasta que termine — algo que un Custom GPT no maneja bien. El
endpoint correcto es `run-sync-get-dataset-items`, que ejecuta y devuelve los ítems en la misma
respuesta.

---

## Esquema corregido

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "Apify Social Listening API",
    "description": "Ejecuta un Actor de Apify que busca menciones en X y devuelve los resultados en la misma llamada.",
    "version": "1.1.0"
  },
  "servers": [
    { "url": "https://api.apify.com/v2" }
  ],
  "paths": {
    "/acts/{actorId}/run-sync-get-dataset-items": {
      "post": {
        "summary": "Buscar menciones en X y devolver los resultados",
        "description": "Ejecuta el Actor de scraping de forma sincrónica y devuelve directamente los posteos encontrados.",
        "operationId": "buscarMenciones",
        "parameters": [
          {
            "name": "actorId",
            "in": "path",
            "required": true,
            "description": "Actor a ejecutar. El separador entre usuario y nombre es '~', no '/'. Ejemplo: apidojo~tweet-scraper",
            "schema": { "type": "string" }
          }
        ],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "searchTerms": {
                    "type": "array",
                    "items": { "type": "string" },
                    "description": "Nombre de la marca o persona a buscar."
                  },
                  "maxItems": {
                    "type": "integer",
                    "description": "Máximo de posteos a devolver. Cada resultado consume crédito de Apify: usar 25 para pruebas, 100 para un informe."
                  }
                },
                "required": ["searchTerms"]
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Los posteos encontrados.",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": { "type": "object" }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

> El cuerpo del request (`searchTerms`, `maxItems`) depende del Actor elegido — cada uno define
> su propio esquema de input. Ajustarlo según lo que quede registrado en
> [`apify-actor-x.md`](apify-actor-x.md).

## Autenticación

En la configuración de la Action: **Authentication → API Key → Bearer**, y pegar el API token
de Apify.

## La limitación de fondo

Compartir el Custom GPT comparte **tu** token de Apify: cada persona que lo use gasta de tu
crédito, y no hay forma de que use el suyo. Con ~$5/mes de crédito gratuito, unas pocas
consultas ajenas lo agotan. Eso, más que el trabajo de armarlo, es la razón por la que esta vía
está diferida — el CLI no tiene ese problema, porque cada quien pone sus propias credenciales.
