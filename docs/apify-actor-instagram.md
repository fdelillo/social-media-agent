# El actor de Instagram

Notas sobre el Actor de Apify que provee las menciones de Instagram, en paralelo a
[`apify-actor-x.md`](apify-actor-x.md). La Fase 1 lee este archivo para saber qué `actorId`
invocar, qué forma tiene su input y cómo mapear su salida a `Mencion`.

**Probado el 2026-09-05** contra la cuenta `fdelillo` (plan FREE), en 14 corridas por $0.17, y
**ampliado el 2026-09-06** con 4 corridas más por $0.05 que corrigieron lo que decía sobre la
ventana temporal.

---

## Lo primero, porque cambia todo lo demás: en Instagram no hay búsqueda de texto

En X, una mención es el resultado de buscar el nombre de la marca en toda la red:
`"DonWeb" -is:retweet lang:es min_faves:5`. **En Instagram esa búsqueda no existe.** El campo
`search` del actor busca *hashtags, perfiles y lugares* — nunca el texto de los captions. No hay
forma, a ningún precio, de preguntar "quién escribió el nombre de mi marca esta semana".

Eso obliga a componer la mención desde tres fuentes distintas, y no son intercambiables:

| Fuente | Qué trae | Estado |
| :--- | :--- | :--- |
| **Comentarios en los posteos propios** | Lo que la gente le escribe a la marca debajo de su contenido | ✅ funciona bien |
| **Etiquetados** (`mentions`) | Posteos de terceros donde la cuenta está etiquetada | ⚠️ techo de 21 y sin filtro de calidad |
| **Hashtag** | Posteos con `#marca` | ❌ roto en plan FREE (ver *Las dos formas de fallar*) |

**La consecuencia de diseño está en la decisión 12:** Instagram entra al producto por los
comentarios en los posteos propios, no por la escucha de red. Es exactamente el bloque *Debajo de
tus posteos* que el informe ya tiene desde la decisión 10, y en Instagram es lo bueno de la
fuente, no un complemento.

---

## Actor elegido

| | |
| :--- | :--- |
| **`actorId`** | `apify~instagram-scraper` |
| **Costo en plan FREE** | $0.0027 por resultado = **$2.70 / 1.000** |
| **Start fee** | ninguno |
| **Funciona por API en gratuito** | ✅ sí, con dos techos (abajo) |
| **Presupuesto que implica** | ~1.850 resultados/mes con los $5 de crédito |

Se eligió porque **un solo actor cubre los tres modos** vía `resultsType`, lo que permite que la
Action de ChatGPT siga siendo una sola operación, igual que la de X con su `mode`. Un actor por
modo habría significado tres Actions y tres esquemas.

### El precio es 18 veces el de X, y ese es el hallazgo que más pesa

| | Actor de X | Actor de Instagram |
| :--- | ---: | ---: |
| Por 1.000 resultados, plan FREE | $0.15 | **$2.70** |
| Un informe de 100 menciones | $0.015 | **$0.27** |
| Informes que entran en los $5 mensuales | ~330 | **~18** |

No es un detalle de tarifa: **es lo que hace que Instagram deje de ser gratis.** El proyecto
entero cabía en el crédito mensual de Apify porque X cuesta centavos. Con Instagram, cuatro
corridas diarias de 100 resultados son ~$32/mes solo de datos — arriba del ~$42/mes del modelo
que ya calculaba `REQUISITOS.md`, y esta vez sin la salida de "vigilar barato".

Como en X, **el precio de la página del actor es el del escalón más alto**; el real está en la
API y hay que consultarlo antes de elegir nada:

```bash
curl -s -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://api.apify.com/v2/acts/apify~instagram-scraper" \
  | python3 -c "import json,sys; print(json.dumps(json.load(sys.stdin)['data']['pricingInfos'][-1], indent=2))"
```

La alternativa barata existe y quedó descartada: ver abajo.

---

## Input

`resultsType` cumple el papel que `mode` cumple en el actor de X, pero **su default (`posts`) sí
sirve**, así que no falla si se lo omite — falla peor: trae otra cosa sin avisar.

```json
{
  "resultsType": "comments",
  "directUrls": [
    "https://www.instagram.com/p/Dc3eEZZjYIl/",
    "https://www.instagram.com/p/Dc1JI40OwKT/"
  ],
  "resultsLimit": 15
}
```

| Campo | Valor | Nota |
| :--- | :--- | :--- |
| `resultsType` | `posts`, `comments`, `mentions`, `reels`, `details` | Cada uno exige una forma de URL distinta |
| `directUrls` | array | `/p/` para posteos y comentarios, `/username/` para perfil y etiquetados |
| `resultsLimit` | entero | **Es por URL, no por corrida** (verificado) |
| `onlyPostsNewerThan` | `"2 days"`, `"2026-09-01"`, ISO | **Filtra el posteo, no el comentario.** Ver abajo |
| `search` + `searchType` | hashtag / profile / place | No sirve acá: no busca texto |

### `resultsLimit` es por URL, y eso simplifica el flujo respecto de X

Verificado: dos URLs de posteo con `resultsLimit: 5` devolvieron **10 comentarios, 5 por cada
una, en una sola llamada** de 2,5 KB. En X, traer las respuestas de tres posteos costaba tres
llamadas a `Get Replies`, una por `id`. Acá los tres posteos entran juntos.

El corolario es que `resultsLimit` multiplica: tres posteos a 15 son 45 resultados facturados,
no 15. Es la forma más fácil de gastar de más.

### La ventana temporal no significa lo mismo que en X

Es la diferencia que más fácil se pasa por alto, porque el parámetro existe y no da error:
**`onlyPostsNewerThan` filtra por la fecha del posteo, nunca por la del comentario.**

Con `resultsType: "comments"` eso quiere decir que descarta el **posteo padre** entero si se
publicó fuera de la ventana, sin mirar un solo comentario. Medido el 2026-09-06: pidiendo una
ventana de 1 hora sobre un posteo del día anterior, el Actor no devolvió "cero comentarios
recientes" sino el error `no_items` — *"Comments for … are private or empty"*.

**La consecuencia es de producto, no de API.** Sobre una marca que postea una vez por semana,
pedir "las últimas 6 horas" devuelve nada, aunque el posteo del martes tenga doscientos
comentarios de esta mañana. Y es justo el caso que más interesa: el reclamo fresco vive debajo de
contenido viejo.

Con `resultsType: "mentions"` o `"posts"` el parámetro sí hace lo que uno espera, porque ahí la
unidad *es* el posteo. Pero deja pasar los fijados: `"2 days"` sobre un perfil devolvió 10
posteos, **ocho dentro de la ventana y dos de hace un mes y seis semanas**. El propio esquema lo
advierte ("pinned posts may still appear even with this filter set").

Así que la ventana se acota distinto según qué se traiga:

| | Cómo se acota |
| :--- | :--- |
| Comentarios | **Sin el parámetro.** Se traen y se descartan por `timestamp` |
| Etiquetados y posteos | Con el parámetro, y además se refiltra por los fijados |

Los comentarios vienen **ordenados del más nuevo al más viejo** (verificado), así que el descarte
por `timestamp` es un corte y no un barrido.

**Y esto rompe media decisión 11.** Aquella se tomó sobre X, donde `since:` acota en origen
precisamente para no pagar por lo que se descarta. En Instagram no hay forma de acotar los
comentarios en origen: se trae, se paga y recién ahí se descarta — a 18 veces el precio de X. La
decisión sigue valiendo para X y para los etiquetados de Instagram; para los comentarios no hay
manera de sostenerla, y el desperdicio es parte del costo.

Queda además un techo que la ventana no levanta: son 15 comentarios por posteo en plan gratuito.
Si en esas 6 horas hubo 200, se ven 15 y **no hay forma de saber cuántos faltaron**.

## Los dos techos del plan FREE, medidos

El esquema del actor los anuncia y las corridas los confirman al número exacto:

| Modo | Pedido | Devuelto |
| :--- | ---: | ---: |
| `comments` sobre un posteo con 62 comentarios | 20 | **15** |
| `mentions` sobre un perfil grande | 25 | **21** |

Los dos se levantan con el plan Starter de Apify. En gratuito son un piso duro, y hay que
declararlo en el informe: sobre un posteo con 62 comentarios estamos leyendo 15.

### Y el techo de `mentions` no es su peor problema

Las 21 menciones que devuelve **no vienen ordenadas ni filtradas por nada.** No hay equivalente
de `min_faves`, ni de `lang`, ni de `-is:retweet`, ni la distinción `Top` / `Latest` que en X
separó los datos buenos de los inservibles. La primera mención de la corrida de prueba fue una
cuenta que etiqueta 26 marcas de golpe para que alguna la repostee, y varias no nombran a la
cuenta en el caption: están etiquetadas en la foto.

Es el mismo problema de calidad que en X resolvió `query_type: "Top"`, pero **acá no hay perilla
que tocar.** Por eso los etiquetados entran al informe como bloque aparte y con la advertencia
puesta, no mezclados en el porcentaje general.

---

## `fields` funciona, y otra vez es lo que hace que esto entre

Es un parámetro de la API de datasets de Apify, no del actor, así que sirve igual que en X. La
diferencia acá es mucho más grande, porque el objeto de Instagram es enorme:

| | Sin `fields` | Con `fields` |
| :--- | ---: | ---: |
| Tres posteos de perfil | 84.008 bytes | **2.276 bytes** |
| Por posteo | 28 KB | **0,76 KB** |
| Campos | 28, con estructuras anidadas | 13 |

**Un recorte de 37×.** Sin `fields`, cien posteos serían 2,8 MB y no hay Action que lo tolere;
con `fields` son ~76 KB, por debajo de los 160 KB que en X ya rozaban el corte.

Los valores a usar, que difieren según el modo:

```
# posts, mentions, reels
id,shortCode,url,caption,ownerUsername,ownerFullName,timestamp,likesCount,commentsCount,type,hashtags,mentions,inputUrl,error,errorDescription

# comments
id,text,ownerUsername,timestamp,likesCount,repliesCount,postUrl,error,errorDescription
```

**Las dos listas terminan en `error,errorDescription`, y no es decorativo.** El recorte se aplica
también a los objetos de error, así que sin esos dos campos una corrida fallida se proyecta a
`[{}]` — cuatro bytes, HTTP 201, indistinguible de un resultado vacío. Es el mismo agujero que en
X hay que tapar con la regla del `id`, salvo que acá se puede evitar de raíz: pidiendo los dos
campos, el motivo del fallo vuelve a viajar en la respuesta.

---

## Salida

El texto de un posteo está en **`caption`**; el de un comentario, en **`text`**. No hay
truncamiento como el `text` / `full_text` de X: ambos vienen completos.

| Campo en la salida | Campo en `Mencion` |
| :--- | :--- |
| `id` | `id_externo` |
| `caption` (posteo) / `text` (comentario) | `texto` |
| `ownerUsername` | `autor` |
| `url` (posteo) / `commentUrl` (comentario) | `url` |
| `timestamp` | `publicado_en` |
| `likesCount`, `commentsCount`, `repliesCount` | `metricas` |

**`timestamp` viene en ISO 8601 con `Z`** (`2026-09-05T10:00:01.000Z`), así que a diferencia de
X no hay que parsear un formato propio. Es una línea menos en la Fase 1.

Para calcular `relacion` (decisión 9), el mapeo de Instagram es:

| `relacion` | En Instagram |
| :--- | :--- |
| `propia` | `ownerUsername` es una cuenta oficial del objetivo |
| `respuesta` | vino de `resultsType: comments` sobre un posteo propio |
| `dirigida` | vino de `resultsType: mentions`, o el objetivo aparece en `mentions` del caption |
| `sobre` | el resto — que en Instagram, sin búsqueda de texto, casi no existe |

Ese último renglón es la diferencia de fondo con X, y hay que decirlo en el informe: **en
Instagram no se ve lo que se dice de vos sin nombrarte.** Se ve lo que te dicen y lo que te
etiqueta.

---

## Las dos formas de fallar, y una es silenciosa

**1. El error estructurado, que es la buena — si no lo recortás vos.** Cuando el handle no existe
o el contenido es inaccesible, el actor devuelve un objeto con `error` y `errorDescription` en vez
del `[{}]` mudo del actor de X. **Pero `fields` se aplica también a ese objeto**, así que una lista
que no incluya esos dos campos lo convierte exactamente en el `[{}]` que el actor de X produce
solo. La ventaja sobre X no está en el actor: está en acordarse de pedir los campos.

Cuando llegan, el error distingue dos casos, y la distinción sirve:

| Respuesta | Qué significa |
| :--- | :--- |
| `"error": "not_found"` — *Post does not exist* | El handle no existe |
| `"error": "no_items"` — *Empty or private data* | Existe, pero es privado o está vacío |

Medido: `mercadolibre` y `donwebok` no existen como cuentas de Instagram; `donweb`,
`mercadolibre.ar` y `mercadolibrearg` existen pero no devolvieron posteos. **Verificar el handle
antes de correr es parte del flujo**, no una comodidad: cada intento fallido se factura igual.

**2. El fallback a Google, que es la peligrosa.** Buscando el hashtag `#mercadolibre` con
`searchType: "hashtag"`, el actor no falló: devolvió HTTP 201 y este objeto, bien formado:

```json
[{ "searchTerm": "#mercadolibre", "searchSource": "google",
   "name": "knimh", "postsCount": 0,
   "url": "https://www.instagram.com/explore/tags/knimh", "id": "knimh" }]
```

Un hashtag inventado, con cero posteos, cobrado. Es la misma trampa que el `[{}]` de X pero peor,
porque el objeto **parece un resultado válido** y un modelo que no lo mire de cerca lo va a
tratar como dato. La regla de descarte es exacta: **si un resultado trae `searchSource`, no es
una mención.** Es la razón por la que la vía de hashtag queda fuera del kit.

**Las dos se facturan.** Cada corrida fallida de las de arriba costó $0.0027 igual que una buena.

---

## El endpoint

```
POST https://api.apify.com/v2/acts/apify~instagram-scraper/run-sync-get-dataset-items?maxTotalChargeUsd=<tope>&fields=<lista>
Authorization: Bearer $APIFY_TOKEN
Content-Type: application/json
```

Idéntico al de X, con el mismo `~` como separador y el mismo `maxTotalChargeUsd` obligatorio —
que acá importa más, porque `resultsLimit` es por URL y multiplica.

Tiempos medidos: 15 comentarios en 9 s, 21 etiquetados en 37 s, 3 posteos en 7 s. Como en X,
**lo domina el arranque del actor y no la cantidad**, y todo queda muy por debajo del corte de
300 s del endpoint síncrono.

---

## Descartados

| Actor | Motivo |
| :--- | :--- |
| [`apidojo/instagram-scraper`](https://apify.com/apidojo/instagram-scraper) | **$0.50 / 1.000, cinco veces más barato, y anda** (verificado con `natgeo`). Descartado porque solo hace posteos por URL: no tiene modo `comments` ni `mentions`, que son las dos fuentes que Instagram necesita. **Vale la pena revisarlo** si alguna vez alcanza con posteos de perfil. |
| [`apify/instagram-comment-scraper`](https://apify.com/apify/instagram-comment-scraper) | Mismo precio ($0.0026) y solo cubre comentarios. El actor general ya los hace. |
| [`apify/instagram-tagged-scraper`](https://apify.com/apify/instagram-tagged-scraper) | Mismo precio ($0.0027), mismo techo de ~20 en gratuito, y solo cubre etiquetados. |
| [`apify/instagram-hashtag-scraper`](https://apify.com/apify/instagram-hashtag-scraper) | La vía de hashtag es la que falla en silencio. Rating 3.39, el más bajo de la familia. |
