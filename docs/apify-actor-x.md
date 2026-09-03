# El actor de X

Notas sobre el Actor de Apify que provee las menciones. La Fase 1 lee este archivo para saber
qué `actorId` invocar, qué forma tiene su input y cómo mapear su salida a `Mencion`.

**Probado el 2026-09-03** contra la cuenta `fdelillo` (plan FREE).

---

## Actor elegido

| | |
| :--- | :--- |
| **`actorId`** | `scrape.badger~twitter-tweets-scraper` |
| **Costo en plan FREE** | $0.00015 por tweet = **$0.15 / 1.000** |
| **Start fee** | ninguno |
| **Funciona por API en gratuito** | ✅ sí, verificado |
| **Presupuesto que implica** | ~33.000 menciones/mes con los $5 de crédito |

### El precio depende del plan, y eso decidió la elección

Es el hallazgo que no estaba en ninguna página de actor: Apify permite **precios por escalón de
plan**, y algunos actores castigan al plan gratuito. Comparando los dos finalistas en el campo
`eventTieredPricingUsd` de la API:

| Actor | FREE | GOLD | Anunciado en su página |
| :--- | :--- | :--- | :--- |
| `xtdata/twitter-x-scraper` | **$5.00 / 1.000** | $0.25 / 1.000 | "$0.25" — que es la tarifa GOLD |
| `scrape.badger/twitter-tweets-scraper` | **$0.15 / 1.000** | $0.12 / 1.000 | "$0.15" |

`xtdata` cobra en gratuito **20 veces** su precio anunciado, más un start fee y un mínimo de $3
por corrida: mil tweets se comerían el presupuesto mensual entero. `scrape.badger` cobra en
gratuito casi lo mismo que en GOLD.

La lección para elegir cualquier actor de acá en adelante: **el precio de la página es el del
escalón más alto**. El real está en la API:

```bash
curl -s -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://api.apify.com/v2/acts/<actorId>" \
  | python3 -c "import json,sys; print(json.dumps(json.load(sys.stdin)['data']['pricingInfos'][-1], indent=2))"
```

---

## Input

`mode` es **obligatorio** y su default es `"Get Tweet by ID"`, que no es lo que queremos. Sin
pasarlo explícitamente la corrida falla con un error de validación.

```json
{
  "mode": "Advanced Search",
  "query": "Netflix -is:retweet lang:es min_faves:5",
  "query_type": "Top",
  "max_results": 100
}
```

| Campo | Valor | Nota |
| :--- | :--- | :--- |
| `mode` | `"Advanced Search"` | Obligatorio. Los otros modos son por ID, retweeters, replies, etc. |
| `query` | keyword + operadores | Acepta la sintaxis de búsqueda avanzada de X |
| `query_type` | `"Top"` | `Top`, `Latest` o `Media`. **Usar `Top`** (ver abajo) |
| `max_results` | entero | Default 1000 — **siempre pasarlo**, o la corrida sale carísima |

### `Top` y no `Latest`: la diferencia es la calidad de los datos

Con `query_type: "Latest"` los resultados son los más nuevos, y en una marca grande eso es
mayormente spam. En la prueba con `Netflix`, de 5 resultados: uno vendiendo cuentas premium,
uno de spam de hashtags, y **los cinco con 0 likes, 0 respuestas y 0 retweets**, en cinco
idiomas distintos. Inservible para medir reputación.

Con `query_type: "Top"` y operadores, los 8 resultados fueron cuentas reales con engagement
real y mezcla natural de sentimiento: elogio al marketing de la marca, bronca por series
canceladas, noticias neutras, y hasta un caso de la marca mencionada solo como recurso retórico.

Operadores que valen la pena en `query`:

| Operador | Para qué |
| :--- | :--- |
| `-is:retweet` | Sin retweets. Un retweet no es una opinión nueva y desbalancea el conteo |
| `lang:es` | Un idioma por corrida. Mezclar idiomas ensucia el análisis de narrativa |
| `min_faves:5` | Piso de engagement: filtra la mayor parte del spam de cuentas nuevas |
| `since:` / `until:` | Acotar la ventana temporal |
| `-from:<handle>` | **Excluir al propio objetivo.** Ver abajo |

### Excluir las cuentas del propio objetivo

En las corridas del 2026-09-03, **el 11% del corpus de MercadoLibre eran posteos de
`@marcos_galperin`** (su fundador) y **el 10% del de Jorge Macri eran de `@jorgemacri`**. No son
menciones *sobre* el objetivo: son el objetivo hablando de sí mismo.

Para un informe de reputación eso distorsiona, y siempre en la misma dirección: nadie publica
mal de sí mismo, así que esos posteos empujan la distribución hacia positivo y neutro. Peor aún,
el prompt clasificador los lee como elogios genuinos de terceros.

Se excluyen en la query, sumando un `-from:` por cada cuenta oficial o vocero:

```
(MercadoLibre OR "Mercado Libre") -is:retweet lang:es min_faves:5 -from:marcos_galperin -from:mercadolibre
```

Identificar esas cuentas es trabajo manual por objetivo, y hay que hacerlo antes de la corrida.
La señal para detectarlas después del hecho es un autor que aparece muchas veces: en las tres
corridas, el autor más repetido de cada corpus fue justamente la cuenta propia.

## Salida

**El campo con el texto completo es `full_text`, no `text`.** `text` trunca a 150 caracteres:
en la prueba, un tweet tenía `text` de 150 y `full_text` de 297. Usar `text` mutilaría el
análisis justo en los posteos largos, que suelen ser los más cargados de opinión.

| Campo en la salida | Campo en `Mencion` |
| :--- | :--- |
| `id` | `id_externo` |
| **`full_text`** | `texto` |
| `username` | `autor` |
| _(construir: `https://x.com/{username}/status/{id}`)_ | `url` |
| `created_at` | `publicado_en` |
| `favorite_count`, `retweet_count`, `reply_count`, `quote_count`, `bookmark_count` | `metricas` |

El actor no devuelve un campo `url`: hay que armarlo con `username` e `id`.

`created_at` viene en formato Twitter (`Thu Sep 03 18:00:43 +0000 2026`), no ISO 8601. Hay que
parsearlo con `%a %b %d %H:%M:%S %z %Y`.

Otros campos disponibles que pueden servir después: `lang`, `is_retweet`, `is_quote_status`,
`user_followers_count`, `user_is_blue_verified`, `media`, `hashtags`, `user_mentions`.

---

## Muestras

En `datos/crudo/` (no versionadas):

| Archivo | Qué es |
| :--- | :--- |
| `netflix-latest-2026-09-03.json` | 5 ítems con `Latest`, sin operadores — el caso malo |
| `netflix-top-2026-09-03.json` | 8 ítems con `Top` + operadores — el caso bueno |

Sirven como fixtures de los tests de parseo de la Fase 1, y el primero además documenta cómo se
ve un resultado inservible.

---

## El endpoint

```
POST https://api.apify.com/v2/acts/{actorId}/run-sync-get-dataset-items?maxTotalChargeUsd=<tope>
Authorization: Bearer $APIFY_TOKEN
Content-Type: application/json
```

Ejecuta y devuelve los ítems en la misma respuesta. En el `actorId`, el separador entre usuario
y nombre es `~`, no `/`: `scrape.badger~twitter-tweets-scraper`.

**Pasar siempre `maxTotalChargeUsd`.** Es un tope duro de gasto por corrida y es la única
protección real contra un `max_results` mal puesto o un actor que se dispara.

Si la corrida falla, el cuerpo trae un `run ID` y el log explica la causa:

```bash
curl -s -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://api.apify.com/v2/actor-runs/<runId>/log"
```

Ojo con el timeout: el endpoint síncrono corta a los 300 segundos. Si una corrida de 100
menciones se acerca a ese límite, hay que bajar el `max_results` o pasar al esquema asíncrono
con polling.

---

## Descartados

| Actor | Motivo |
| :--- | :--- |
| [`apidojo/tweet-scraper`](https://apify.com/apidojo/tweet-scraper) | En plan gratuito permite 5 corridas mensuales de 10 ítems **y prohíbe el acceso por API**. Ni el MCP ni el CLI pueden invocarlo. |
| [`xtdata/twitter-x-scraper`](https://apify.com/xtdata/twitter-x-scraper) | $5.00 / 1.000 en plan gratuito: 20× su precio anunciado, más start fee y un mínimo de $3 por corrida. |
| [`kaitoeasyapi/...cheapest`](https://apify.com/kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest) | No se llegó a probar: `scrape.badger` ya cumplía. Su página se contradice sobre el precio ($0.18 vs $0.25). |
