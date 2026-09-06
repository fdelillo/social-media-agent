# Reemplazar a Apify: qué implicaría

Análisis del 2026-09-05. Nadie lo pidió por un problema concreto todavía, así que la primera
parte del trabajo es separar **qué nos da Apify** de **qué creemos que nos da**. Sin eso, la
comparación de precios engaña.

**Respuesta corta:** sí, existe la posibilidad, y el reemplazo directo es barato de hacer
(medio día). Pero **no conviene hacerlo por plata**, porque Apify hoy nos sale $0 y el costo del
proyecto automatizado no está en los datos: está en el modelo. Conviene hacerlo solo si el
motivo es sacarse de encima un intermediario que falla en silencio, o si el plan gratuito se
vuelve insuficiente.

---

## Qué nos da Apify, además de los datos

Esto es lo que hay que reponer, no solamente "los tweets":

| Lo que da | Por qué importa acá |
| :--- | :--- |
| **$5/mes de crédito gratis** | Cubre ~33.000 menciones. Es todo nuestro consumo, incluso automatizado. |
| **Un POST síncrono que devuelve el JSON** | `run-sync-get-dataset-items`. Es lo que hace que la Action de ChatGPT sea **una sola llamada**. |
| **`fields`** | Recorta de 41 campos a 10: 45 KB en vez de ~200 KB por corrida. Sin eso el kit de ChatGPT no entra. |
| **`maxTotalChargeUsd`** | Tope duro de gasto por corrida. Es el único freno real ante un `max_results` mal puesto. |
| **Paginado resuelto** | Pedís 100 y llegan 100. No hay cursor que seguir. |
| **Un servidor MCP** | La vía de uso en Claude, sin escribir código. |

Los últimos cuatro no son "comodidad de Apify": son exactamente las piezas que sostienen la
decisión 1 (sin infraestructura) y el requisito 2 (que otro lo use sin instalar nada).

---

## Las cuatro familias de reemplazo

### A. Otro proveedor de datos de X, por API directa

El candidato es [`twitterapi.io`](https://twitterapi.io): `$0.15 / 1.000 tweets`, autenticación
con un header `x-api-key`, sin suscripción. Tiene `GET /twitter/tweet/advanced_search` y
`GET /twitter/tweet/replies`, que son las dos llamadas que el agente usa hoy (decisiones 10 y 11).

**El precio es idéntico al del actor, al centavo: $0.00015 por tweet.** El actor
`scrape.badger` cobra exactamente eso en plan FREE y su descripción anuncia la misma superficie
—advanced search, replies, retweeters, quotes, sin API key, sin rate limits—. No está confirmado,
pero todo indica que **le estamos pagando a un revendedor lo mismo que costaría el mayorista**.

Lo que se gana y lo que se pierde:

| | Apify hoy | twitterapi.io |
| :--- | :--- | :--- |
| Precio por 1.000 | $0.15 | $0.15 |
| Crédito mensual gratis | **$5** | ~$1 de prueba, una vez |
| Llamadas para traer 100 menciones | **1** | ~5 (20 por página, con cursor) |
| Recorte de campos | **`fields`** | no documentado |
| Tope de gasto por corrida | **`maxTotalChargeUsd`** | no |
| Falla en silencio con `[{}]` | sí, medido | un intermediario menos |
| Ventana temporal | `since:AAAA-MM-DD_HH:MM:SS_UTC` | **`since_time:` en epoch** — el formato nuestro está explícitamente no soportado |

**Los dos renglones que más duelen son el paginado y `fields`.** El kit de ChatGPT vive de que
una corrida sea una llamada de 45 KB. Con cursor de 20 en 20 y el objeto completo del tweet,
la misma corrida son cinco llamadas y varias veces el volumen: el GPT tendría que iterar, y eso
es justo donde una Action se rompe.

**Riesgo abierto que hay que verificar antes de cualquier cosa:** la documentación de
`advanced_search` no menciona `lang:`, `min_faves:` ni `-is:retweet`. Toda nuestra estrategia de
query depende de esos tres operadores (`docs/apify-actor-x.md`): sin `min_faves:5` volvemos al
corpus de spam que descartó `Latest`. Se comprueba con una corrida de 5 ítems y el crédito de
prueba.

Alternativas del mismo tipo, por si el primero falla: [SocialData](https://socialdata.tools)
($0.20/1.000, facturado por ítem entregado) y otros proveedores que se anuncian más baratos
($0.05/1.000), sin verificar.

### B. La API oficial de X

**Posible, y 33 veces más cara.** Desde febrero de 2026 no hay plan gratuito ni Basic para
cuentas nuevas: es pago por uso, **$0.005 por post leído**, con tope de 2 millones de lecturas
mensuales y Enterprise (~$42.000/mes) por encima.

| | Apify | X oficial |
| :--- | ---: | ---: |
| Un informe de 100 menciones | $0.015 | **$0.50** |
| Automatizado, 4 corridas/día | $1.80/mes | **$60/mes** |

Se justifica solo si aparece un requisito de cumplimiento —usar datos con licencia, no
scrapeados—. Como decisión técnica no tiene defensa a este volumen.

### C. Cambiar de fuente, no de proveedor

Bluesky y Mastodon tienen API pública gratuita y sin scraping de por medio. Reddit y Google News
por RSS también. **Esto no reemplaza a Apify: reemplaza a X.** Para medir reputación de una marca
argentina, la conversación no está ahí, así que resuelve el problema de costo destruyendo el
producto. Vale como *fuente adicional* algún día (la decisión 6 ya deja el punto de extensión
abierto), no como reemplazo.

### D. Scraping propio

`twscrape` y similares corren con cuentas propias de X y salen $0 en dinero. Cuestan en todo lo
demás: violan los términos de servicio, la cuenta se banea, se rompen cuando X cambia algo, y
**necesitan un proceso propio prendido con cookies guardadas**. Eso choca de frente con la
decisión 1 y hace inentregable el requisito 2: nadie va a "instalarse" el agente si el primer
paso es prestar su cuenta de X. Nitter, que era la salida elegante, ya no existe como servicio
confiable.

---

## Por qué hoy la respuesta es "no ahora"

El número que ordena todo está en `REQUISITOS.md` y no cambió:

| Costo mensual del proyecto automatizado (cada 6 h) | |
| :--- | ---: |
| Datos (Apify, 12.000 menciones) | ~$1,80 — **dentro de los $5 gratuitos** |
| **El modelo** (~$0,35 × 120 corridas) | **~$42** |

**Apify es la parte gratis del proyecto.** Migrar a twitterapi.io al mismo precio por tweet nos
haría *empezar a pagar* $1,80/mes que hoy no pagamos, y no tocaría los $42 que sí duelen. La
pregunta de costo que sigue abierta es la primera de `REQUISITOS.md` —vigilancia barata y
análisis caro solo ante un pico—, y esa se responde sin cambiar de proveedor de datos.

### Los tres motivos que sí lo justificarían

1. **El actor falla en silencio.** Devolvió `[{}]` con HTTP 201 en vez de un error (medido el
   2026-09-04). Ir directo al mayorista saca una capa de la que no controlamos nada. Hoy se
   compensa con reintentos en las instrucciones; si eso resulta insuficiente en uso real, el
   motivo pasa a ser bueno.
2. **El actor desaparece o cambia de precio.** Es de un tercero. `docs/apify-actor-x.md` ya
   documenta que el precio anunciado no es el que se paga y que varía por plan.
3. **Se agota el crédito.** Con varios objetivos o ventanas más agresivas, $5 se termina. Ahí
   Apify deja de ser gratis y la comparación se vuelve pareja — y gana quien no cobre margen.

### El seguro barato, que sí conviene hacer ya

No migrar, sino **dejar de depender del proveedor en el código que todavía no está escrito**.
La Fase 1 va a tener un cliente de datos: que la forma de `Mencion` sea la frontera y que el
proveedor entre por detrás, como ya está previsto para Instagram en la decisión 6. Con eso,
cambiar de Apify a twitterapi.io es reescribir un archivo, no el agente.

Lo que **no** se puede blindar igual es el kit de ChatGPT: ahí el proveedor está incrustado en el
esquema de la Action, y cambiarlo es reescribir el kit y pedirle a quien lo usa que rearme el GPT.
Esa es la parte cara de la migración, y es la razón principal para no hacerla mientras el kit
esté esperando feedback.

---

## Qué habría que hacer, si se decide migrar

En orden, y las dos primeras cuestan una corrida de prueba cada una:

1. Sacar la clave de prueba de twitterapi.io y verificar **los tres operadores** (`lang:`,
   `min_faves:`, `-is:retweet`) y `since_time:` con 5 ítems.
2. Medir el paginado real de `advanced_search`: cuántos tweets por página y cuántas llamadas
   para 100.
3. Reescribir el mapeo de salida: los campos cambian de nombre (`full_text` → `text`,
   `favorite_count` → `likeCount`, `created_at` sigue sin ser ISO) y `url` viene dado.
4. Rehacer `chatgpt/accion-apify.json` — nombre incluido — y la guía publicada, que vive en un
   link que hay que actualizar, no recrear (ver `ESTADO.md`).
5. Reponer a mano lo que se pierde: tope de gasto y recorte de campos pasan a ser
   responsabilidad nuestra.

## Fuentes

- [X API pay-per-use, 2026](https://twitterapi.io/blog/x-api-cost-breakdown-2026) y
  [tiers 2026](https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/)
- [twitterapi.io](https://twitterapi.io/) — [docs](https://docs.twitterapi.io/introduction),
  [advanced search](https://docs.twitterapi.io/api-reference/endpoint/tweet_advanced_search),
  [replies](https://docs.twitterapi.io/api-reference/endpoint/get_tweet_reply)
- [SocialData](https://socialdata.tools/)
- Precio del actor: consultado por API el 2026-09-05 sobre `scrape.badger~twitter-tweets-scraper`
