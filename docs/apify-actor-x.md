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

## Candidatos (relevados en septiembre 2026)

Los precios cambian y los actores aparecen y desaparecen; verificar en Apify antes de decidir.

| Actor | Precio / 1.000 | Nota |
| :--- | :--- | :--- |
| [`apidojo/tweet-scraper`](https://apify.com/apidojo/tweet-scraper) | ~$0.40 | Popular y mantenido. Buen primer candidato. |
| [`xtdata/twitter-x-scraper`](https://apify.com/xtdata/twitter-x-scraper) | ~$0.50 | El más caro de la lista. |
| [`kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest`](https://apify.com/kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest) | ~$0.18 | Verificar si exige plan pago. |
| [`scrape.badger/twitter-tweets-scraper`](https://apify.com/scrape.badger/twitter-tweets-scraper) | ~$0.15 | El precio publicado aplica **en planes pagos**. |

**Criterio de elección, en orden:**

1. **Que funcione en el plan gratuito.** Varios de los baratos no.
2. **Que devuelva el texto completo.** Un actor que trunca el copy vuelve inútil el análisis de
   sentimiento, por barato que sea.
3. **Que acepte búsqueda por keyword**, no solo por handle o por URL de perfil. El caso de uso
   es "quién habla de esta marca", no "qué publicó esta cuenta".
4. Recién después, el precio.

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
