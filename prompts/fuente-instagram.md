# La fuente de Instagram

Este archivo es el flujo completo de Instagram. Las instrucciones del agente lo nombran y no lo
repiten, porque el campo de Instrucciones de ChatGPT admite 8.000 caracteres y ya está lleno con
el flujo de X.

Todo lo de acá está medido contra el Actor real el 2026-09-05. El detalle técnico y los números
están en `docs/apify-actor-instagram.md`.

---

## Lo primero, y decilo cuando corresponda: en Instagram no se puede buscar texto

En X buscás el nombre de la marca en toda la red. **En Instagram eso no existe.** No hay ninguna
forma —a ningún precio, con ningún Actor— de preguntar quién escribió el nombre de una marca en
un posteo. La búsqueda de Instagram encuentra hashtags, perfiles y lugares, nunca el texto de un
caption.

Si el usuario pide "buscá menciones de la marca en Instagram" esperando lo mismo que en X,
**decíselo antes de gastar una corrida**, y ofrecé lo que sí se puede: los comentarios en sus
posteos y los posteos donde lo etiquetaron.

## Las tres fuentes, y cuál usar

| Fuente | `resultsType` | Estado |
| :--- | :--- | :--- |
| Comentarios en los posteos propios | `comments` | La principal. Es la que anda bien. |
| Posteos donde etiquetaron a la cuenta | `mentions` | Secundaria, con reservas. |
| Hashtag | — | **No usar.** Falla en silencio (abajo). |

# Paso A — Pedir el handle, y verificarlo

Preguntá el **usuario de Instagram** del objetivo, sin la arroba. No lo adivines a partir del
nombre ni del handle de X: son distintos más veces de las que uno espera. Medido: `mercadolibre`
y `donwebok` no existen como cuentas de Instagram.

Una corrida contra un handle que no existe **se factura igual que una buena**, así que la primera
llamada conviene que sea barata: `resultsType: "posts"` con `resultsLimit: 3`. Eso confirma el
handle y de paso te devuelve los posteos con su `commentsCount`, que es lo que necesitás para el
paso B.

Si la respuesta trae `error`, leelo, porque distingue dos casos:

- `not_found` (*Post does not exist*) → **el handle no existe.** Pedí el correcto.
- `no_items` (*Empty or private data*) → **existe pero es privado o no tiene posteos.** No
  insistas: no hay nada que traer y cada intento cuesta.

# Paso B — Los comentarios en los posteos propios

Es la fuente principal de Instagram y donde está lo accionable: clientes reclamando debajo del
contenido de la propia marca.

1. Del paso A, tomá **los tres posteos con más `commentsCount`**.
2. Llamá **una sola vez** a `buscarInstagram` con `resultsType: "comments"`, pasando las tres
   URLs juntas en `directUrls` y `resultsLimit: 15`.

`resultsLimit` es **por URL**, así que esa llamada trae hasta 45 comentarios y cuesta ~$0,12.
No hace falta una llamada por posteo.

Acordate de cambiar el parámetro `fields` al valor de comentarios; con el de posteos volverían
todos los campos vacíos. **Y no le saques `error,errorDescription`**, que van al final de las dos
listas: sin esos dos campos, una corrida fallida se recorta a `[{}]` y no hay forma de saber que
falló (paso D).

**El techo de 15 comentarios por posteo es del plan gratuito de Apify y no se puede pasar.**
Declaralo en el informe: si el posteo declara 62 comentarios y trajiste 15, decí "15 de 62".

Estos comentarios llevan `relacion: respuesta` y van al bloque **Debajo de tus posteos**.

# Paso C — Los etiquetados, si el usuario los quiere

`resultsType: "mentions"` con la URL del perfil y `resultsLimit: 21`. Traen los posteos de
terceros donde la cuenta está etiquetada.

**Advertí antes de correrlo que la calidad es baja, y por qué.** A diferencia de X, acá no hay
ningún filtro: no existe `min_faves`, ni `lang`, ni la distinción entre `Top` y `Latest`. Los 21
resultados que llegan son arbitrarios y buena parte son cuentas promocionales que etiquetan
decenas de marcas de golpe para que alguna las repostee. En la corrida de prueba, la primera
mención etiquetaba 26 cuentas a la vez.

Por eso: **descartá los posteos que etiqueten más de cinco cuentas distintas.** No son menciones,
son redes de reposteo.

El techo de 21 también es del plan gratuito. Estos van con `relacion: dirigida`, al bloque
**Te etiquetaron**.

# Paso D — Antes de usar cualquier respuesta, filtrala

Tres reglas, y las tres son por fallas medidas:

1. **Si un resultado trae el campo `searchSource`, no es una mención: descartalo.** Es el Actor
   cayendo a un fallback de Google, que devuelve un hashtag inventado con HTTP 201 y cara de
   resultado válido. En la prueba, buscar `#mercadolibre` devolvió un hashtag llamado `knimh` con
   cero posteos. Es la falla más peligrosa de esta fuente porque no parece una falla.
2. **Si un resultado trae `error`, no es una mención: descartalo** y leé el motivo (paso A).
   Y si te llega **`[{}]`** —un objeto vacío, HTTP 201— es el mismo error con los campos
   recortados: alguien sacó `error,errorDescription` de la lista de `fields`. Tratalo como falla,
   nunca como resultado.
3. **Si pediste una ventana temporal, refiltrá por `timestamp`.** Es obligatorio, y el porqué
   está en la sección siguiente.

# La ventana temporal, que NO funciona como en X

Si te piden "las últimas 6 horas", **no traduzcas eso a `onlyPostsNewerThan` como harías en X.**
Significa otra cosa y te va a dar un resultado vacío que parece un dato.

**`onlyPostsNewerThan` filtra por la fecha del posteo, no por la de los comentarios.** Con
`resultsType: "comments"` descarta el posteo entero si se publicó fuera de la ventana, sin mirar
un solo comentario. Medido: pidiendo una ventana de 1 hora sobre un posteo de ayer, el Actor no
devolvió "cero comentarios recientes" — devolvió el error `no_items`, *"Comments are private or
empty"*.

La consecuencia importa: **sobre una marca que postea una vez por semana, pedir "las últimas 6
horas" devuelve nada**, aunque tenga doscientos comentarios frescos colgando del posteo del
martes. Que es justo el caso que más interesa.

Entonces, según qué estés trayendo:

| | Cómo se acota la ventana |
| :--- | :--- |
| **Comentarios** (paso B) | **Sin `onlyPostsNewerThan`.** Traelos y descartá por `timestamp` |
| **Etiquetados** (paso C) | `onlyPostsNewerThan` sirve, pero los posteos fijados se cuelan: refiltrá igual |

Los comentarios **vienen ordenados del más nuevo al más viejo** (verificado), así que descartar
por `timestamp` es seguro: cuando encontrás el primero que quedó fuera de la ventana, los que
siguen también están fuera.

**Decí siempre dos cosas cuando uses ventana en Instagram**, porque el lector no las puede
adivinar:

1. **Que se paga lo que se descarta.** A diferencia de X, acá la ventana no se acota en origen.
   Se trae y se filtra, y lo filtrado ya se cobró — a 18 veces el precio de X.
2. **Que el techo de 15 puede estar escondiendo cosas.** Si en esas 6 horas hubo 200 comentarios,
   viste 15 y no hay forma de saber cuántos faltaron. Si trajiste 15 y los 15 entran en la
   ventana, avisá que probablemente haya más.

# Paso E — Clasificar y redactar

Los criterios de `analista-sentimiento.md` y el formato de `redactor-informe.md` valen igual, sin
cambios. Instagram solo cambia de dónde sale cada cosa:

| `relacion` | En Instagram |
| :--- | :--- |
| `propia` | La publicó la cuenta oficial. No es una mención, no se cuenta. |
| `respuesta` | Vino del paso B. Va al bloque *Debajo de tus posteos*. |
| `dirigida` | Vino del paso C. Va al bloque *Te etiquetaron*. |
| `sobre` | **Prácticamente no existe en Instagram**, y hay que decirlo. |

El texto de un comentario está en `text`; el de un posteo, en `caption`.

**La advertencia que el informe de Instagram siempre lleva**, en una línea, porque sin ella el
lector va a creer que vio más de lo que vio:

> En Instagram no se puede buscar por texto, así que este informe no ve lo que se dice de vos sin
> nombrarte: solo lo que te comentan y lo que te etiqueta.

Y si el informe combina X con Instagram, **los porcentajes de cada red van por separado**. No los
promedies: en X predomina lo que se dice *sobre* el objetivo y en Instagram lo que se le dice *a*
él, que es una población distinta y sesgada hacia el reclamo. Mezclarlas produce un número que no
mide nada.

# Lo que cuesta, para decirlo si el usuario pregunta

Instagram cuesta **18 veces** lo que X: $2,70 por 1.000 resultados contra $0,15. Un informe de
Instagram (3 posteos × 15 comentarios + 21 etiquetados) son ~66 resultados, unos **$0,18**; el
mismo informe en X cuesta $0,015. Con los $5 mensuales de Apify entran unos 18 informes de
Instagram, contra unos 330 de X.

Si el usuario va a correr esto seguido, decíselo antes y no después.
