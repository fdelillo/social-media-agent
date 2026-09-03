# El actor de X

Notas sobre el Actor de Apify que provee las menciones. La Fase 1 lee este archivo para saber
qué `actorId` invocar y qué forma tiene su input.

> **Pendiente de completar en la Fase 0, paso 2.** Los candidatos están abajo; la sección
> "Actor elegido" se llena después de probarlos.

---

## Actor elegido

| | |
| :--- | :--- |
| **`actorId`** | _(pendiente)_ |
| **Costo por 1.000 resultados** | _(pendiente)_ |
| **Start fee por corrida** | _(pendiente)_ |
| **Requiere plan pago** | _(pendiente)_ |
| **Probado el** | _(pendiente)_ |

### Forma del input

```json
{ }
```

### Forma de la salida

Qué campo trae el texto, cuál el id, cuál la fecha, cuál el autor y qué métricas expone. Esto
es lo que `fuentes/x.py` va a mapear a `Mencion` en la Fase 1.

| Campo en la salida | Campo en `Mencion` |
| :--- | :--- |
| _(pendiente)_ | `id_externo` |
| _(pendiente)_ | `texto` |
| _(pendiente)_ | `autor` |
| _(pendiente)_ | `url` |
| _(pendiente)_ | `publicado_en` |
| _(pendiente)_ | `metricas` |

### Observaciones de la prueba

- ¿El texto viene completo o truncado?
- ¿Las fechas son recientes?
- ¿Cuántos de los resultados son retweets o spam?
- ¿Qué pasa con un objetivo de bajo volumen — devuelve pocos, o falla?

---

## Candidatos (relevados el 2026-09-03)

**Hallazgo que condiciona todo lo demás: los actores de X más populares restringen el plan
gratuito**, y el precio por 1.000 tweets resultó ser el criterio menos importante. Lo que
decide es si el actor corre por API en gratuito.

| Actor | Precio / 1.000 | Plan gratuito | Input de búsqueda |
| :--- | :--- | :--- | :--- |
| [`apidojo/tweet-scraper`](https://apify.com/apidojo/tweet-scraper) | $0.40, sin start fee | ❌ **Descartado** | `searchTerms` |
| [`kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest`](https://apify.com/kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest) | $0.18 o $0.25 (su propia página se contradice) | ⚠️ Restringido, recomienda plan pago | `twitterContent`, `searchTerms` |
| [`xtdata/twitter-x-scraper`](https://apify.com/xtdata/twitter-x-scraper) | $0.25 | ❓ No documentado | `searchTerms` |
| [`scrape.badger/twitter-tweets-scraper`](https://apify.com/scrape.badger/twitter-tweets-scraper) | $0.15 en planes pagos, + start fee | ❓ No documentado | `query` (sintaxis avanzada de X), `query_type`, `max_results` |

**`apidojo/tweet-scraper` queda descartado** y conviene registrar por qué, para no volver sobre
él: en plan gratuito permite 5 corridas mensuales de 10 ítems cada una **y prohíbe el acceso por
API**. No es que rinda poco — es que ni el MCP ni el CLI pueden invocarlo, porque ambos entran
por la API.

Los dos últimos no documentan su política para el plan gratuito. Eso no es una confirmación de
que funcionen: es una pregunta abierta que solo se responde probando (ver "Cómo probar" abajo).

**Criterio de elección, en orden:**

1. **Que corra por API en el plan gratuito.** Es eliminatorio y no siempre está documentado.
2. **Que devuelva el texto completo.** Un actor que trunca el copy vuelve inútil el análisis de
   sentimiento, por barato que sea.
3. **Que acepte búsqueda por keyword**, no solo por handle o por URL de perfil. El caso de uso
   es "quién habla de esta marca", no "qué publicó esta cuenta".
4. Recién después, el precio.

### Cómo probar

Con el MCP conectado, correr cada candidato con el límite más chico que acepte (5–10 ítems) y
una keyword de volumen alto. Lo que se busca no es la data: es si **la corrida arranca**. Un
error de autorización o de cuota responde la pregunta 1 al instante y sin gastar casi nada.

Cuidado con `scrape.badger`: su documentación aclara que **las corridas que devuelven cero
resultados se cobran igual**, porque la request al upstream se hace de todos modos. Usar una
keyword que con seguridad tenga menciones.

### Si ninguno funciona en gratuito

Es un desenlace posible y hay que decidirlo, no descubrirlo a mitad de la Fase 1. Las opciones,
de menor a mayor compromiso:

1. Buscar en la [store de Apify](https://apify.com/store?search=twitter) otro actor de X, con el
   filtro puesto en el plan gratuito.
2. Correr la Fase 0 con las corridas mínimas que el plan gratuito permita, aunque den 10 ítems:
   alcanzan para verificar la **forma** de los datos, aunque no para el set dorado de 30.
3. Pagar un mes del plan Starter de Apify para atravesar la Fase 0, y volver a gratuito después.
4. Cambiar de fuente. La API oficial de X tiene su propio costo y sus propios límites, pero es
   una alternativa real si el scraping de X resulta impracticable en gratuito.

---

## El endpoint

Para el CLI de la Fase 1, la llamada correcta es la síncrona, que ejecuta y devuelve los ítems
en una sola request:

```
POST https://api.apify.com/v2/acts/{actorId}/run-sync-get-dataset-items
Authorization: Bearer $APIFY_TOKEN
Content-Type: application/json

{ ...el input del actor... }
```

En el `actorId`, el separador entre usuario y nombre es `~`, no `/`:
`apidojo~tweet-scraper`.

> El documento de visión original propone `POST /actor-tasks/{taskId}/runs`. Ese endpoint
> arranca la corrida y devuelve un objeto *run*: hay que hacer polling del dataset después. Para
> un cliente de una sola llamada, `run-sync-get-dataset-items` es el correcto. El documento
> también apunta a `https://apify.com` como servidor; la API vive en `https://api.apify.com/v2`.

Ojo con el timeout: una corrida de 100 menciones puede tardar bastante, y el endpoint síncrono
corta a los 300 segundos. Si se llega a ese límite, hay que bajar el `--limite` o pasar al
esquema asíncrono con polling.
