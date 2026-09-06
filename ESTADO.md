# Dónde quedamos

Última sesión: **2026-09-06**. Fase 0 en curso, sin código todavía — pero el agente ya se
puede usar: montado en un cliente de chat, sin CLI. Cubre **X e Instagram**.

## Lo que está hecho

| Paso | Estado |
| :--- | :--- |
| 1. Cuenta y token de Apify | ✅ `.env` con `APIFY_TOKEN`, plan FREE |
| 2. Elegir el actor de X | ✅ `scrape.badger~twitter-tweets-scraper`, $0.15/1.000, validado por API |
| 3. Conectar el MCP | ⏭️ opcional, no está en el camino crítico (se usa `curl`) |
| — Vía de uso sin código | ✅ [`chatgpt/`](chatgpt/): un GPT propio con Action, probado contra el Actor |
| 4. Tres corridas | ✅ 300 menciones: MercadoLibre, DonWeb, Jorge Macri — en `datos/crudo/` |
| 5. Set dorado | ✅ 30 etiquetadas, más 4 rondas de medición |
| 6. Medir y ajustar | ⏸️ pausado a propósito: 4 rondas sin converger (ver abajo) |
| 7. Informe de ejemplo | ✅ [`reportes/ejemplo-donweb.md`](reportes/ejemplo-donweb.md), 100 menciones |
| 8. Segunda fuente: Instagram | ✅ actor elegido y medido, Action y prompts extendidos — sin estrenar |

Gasto de Apify hasta ahora: **~$0.43** de los $5 mensuales (300 menciones de los corpus, 16
corridas de prueba del kit de X, 13 corridas de Instagram por $0.18, y 4 más por $0.05 para
verificar la ventana temporal).

## Lo que pasa mañana

**Nada, hasta que llegue feedback de uso real.** El kit de ChatGPT está terminado y compartido:
[`chatgpt/`](chatgpt/) más una guía de puesta en marcha publicada como página. Alguien lo va a
armar con su propio token y usarlo; lo que devuelva decide qué se toca después.

> ✅ **Las dos páginas se republicaron el 2026-09-06** y están al día con el repo. La guía suma
> una sección para quien ya armó el GPT antes (no se le rompe nada; son tres cambios de dos
> minutos) y otra que hace explícitas las decisiones que salieron del análisis. El informe suma
> *Qué decidió lo que ves acá*, con los recortes que ya estaban operando sin declararse.
>
> ⚠️ **Republicar no cambia lo que ve quien ya tenía el link.** La guía está compartida con una
> versión fijada: hay que **mover el share pin** desde el menú de la página para que los que ya
> la abrieron vean la nueva.

### Lo publicado, con su link

Estas páginas **no se pueden reconstruir desde el repo**: republicar el mismo archivo desde otra
conversación crea una página nueva con otra URL. Si hay que actualizarlas, se actualizan sobre
estos links.

| Qué | Link | Se genera desde |
| :--- | :--- | :--- |
| Guía de puesta en marcha en ChatGPT | https://claude.ai/code/artifact/5251dedc-ca98-4e92-9a05-88c772274d9a | `chatgpt/README.md` + `chatgpt/instrucciones.md` + `chatgpt/accion-apify.json` |
| Informe de ejemplo, versión leíble | https://claude.ai/code/artifact/7675a180-e5ae-4599-a1a6-8b3033a69989 | `reportes/ejemplo-donweb.md` |

Las dos nacen privadas: para que alguien las abra hay que compartirlas desde el menú de la
página. La guía es la que se entrega; el informe es la referencia de formato.

Las tres preguntas que se le pidieron: si las recomendaciones eran ejecutables o relleno, si
alguna mención quedó mal clasificada y cuál, y qué sección leería primero con dos minutos.

**Lo único sin probar del kit** es la Action desde dentro de ChatGPT, por no tener cuenta paga
acá. La API responde como el esquema dice, en tiempos y tamaños que una Action tolera, pero el
armado del GPT está sin estrenar. Si algo falla va a ser en el paso 4 de la guía.

**El ciclo de medición queda pausado, y es una decisión, no un olvido.** Cuatro rondas dieron
67%, 75%, 60% y (65% / 70%) sin converger, y con 20 menciones por ronda el intervalo de confianza
al 95% es de ±20 puntos: el 80% del criterio de salida cae dentro del intervalo de casi todas las
mediciones hechas. Una ronda de 20 no puede demostrar que se pasó ni que no. Seguir agregando
rondas era comprar ruido.

> ⚠️ **El informe de ejemplo quedó atrás de los prompts.** `reportes/ejemplo-donweb.md` es
> anterior a las decisiones 9 y 10 —`relacion`, los dos modos, la sección *Quién te está
> hablando* partida en dos bloques— y no las refleja. Regenerarlo está pendiente a propósito:
> primero conviene saber si el formato nuevo convence a alguien que lo use.

## Instagram, agregado el 2026-09-05

Se sumó como segunda fuente, medido en 13 corridas reales por $0.18. Está en
[`docs/apify-actor-instagram.md`](docs/apify-actor-instagram.md) y en la decisión 12.

**El hallazgo que ordena todo lo demás: en Instagram no existe la búsqueda por texto.** No hay
forma, a ningún precio ni con ningún actor, de preguntar quién escribió el nombre de una marca —
el buscador de Instagram encuentra hashtags, perfiles y lugares, nunca el texto de un caption. La
pregunta que estructura la fuente de X no se puede formular. Así que Instagram entra por los
**comentarios en los posteos propios**, y los etiquetados quedan como bloque secundario.

| Lo medido | |
| :--- | :--- |
| Actor | `apify~instagram-scraper`: un solo actor cubre comentarios, etiquetados y posteos |
| Precio en plan FREE | **$2,70 / 1.000 = 18× el actor de X.** ~18 informes por mes contra ~330 |
| Techo de comentarios | 15 por posteo (pedí 20, llegaron 15) |
| Techo de etiquetados | 21 por cuenta (pedí 25, llegaron 21) |
| `fields` | 84 KB → 2,3 KB por tres posteos: **37×**. Sin él, 100 posteos son 2,8 MB. Tiene que incluir `error,errorDescription` |
| `resultsLimit` | **es por URL**, así que multiplica: 3 posteos × 15 = 45 resultados facturados |
| Ventana temporal | **No es la de X.** Filtra por la fecha del posteo, no la del comentario |

**Dos formas de fallar, y una es peor que el `[{}]` de X.** Cuando el handle no existe el actor
devuelve un error estructurado, que es la buena. La mala es la búsqueda por hashtag: cae a un
fallback de Google y devuelve **un hashtag inventado con HTTP 201 y cara de resultado válido**
(buscando `#mercadolibre` devolvió uno llamado `knimh` con cero posteos). Por eso la vía de
hashtag quedó fuera del kit, y la regla de descarte es: si trae `searchSource`, no es una mención.

**Lo que costó de más:** las instrucciones del GPT llegaron al límite. El campo admite 8.000
caracteres y ya usaba 7.849, así que el flujo de Instagram no entraba. Se resolvió como el
proyecto ya resolvía lo mismo —un tercer archivo de Conocimiento,
[`prompts/fuente-instagram.md`](prompts/fuente-instagram.md)— más compresión de prosa en las
instrucciones para hacerle lugar al enganche. **Quedan 15 caracteres de margen:** lo próximo que
haya que agregar ahí no entra sin sacar algo.

### Corregido el 2026-09-06: la ventana temporal no era la de X

La primera versión de todo esto daba por hecho que "las últimas 6 horas" funcionaba en Instagram
como en X. **No funciona**, y comprobarlo en 4 corridas corrigió tres archivos.

**`onlyPostsNewerThan` filtra por la fecha del posteo, no por la del comentario.** Pedir una
ventana de 1 hora sobre un posteo de ayer no devolvió "cero comentarios recientes": devolvió el
error `no_items`. Descartó el posteo padre entero sin mirar un comentario. Sobre una marca que
postea una vez por semana, "las últimas 6 horas" devuelve nada aunque el posteo del martes tenga
doscientos comentarios de hoy — que es justo el caso que interesa, porque el reclamo fresco
cuelga de contenido viejo. Los comentarios se acotan trayéndolos sin ventana (vienen del más
nuevo al más viejo) y descartando por `timestamp`.

**Pega en el requisito 4.** La cadencia de 6 horas se sostenía en que la ventana se acota en
origen: sin superposición, sin deduplicar, sin base de datos. En los comentarios de Instagram eso
no se puede hacer, así que hay que **pagar cuatro veces por día por resultados que se tiran**, a
18× el precio de X. La decisión 11 sigue entera para X y para los etiquetados; para los
comentarios de Instagram no hay forma de sostenerla.

**Y un defecto propio, que es la lección más transferible:** el recorte de `fields` se aplica
también a los objetos de error. La lista original no pedía `error`, así que convertía cualquier
fallo en `[{}]` —el mismo objeto mudo del actor de X— y volvía inaplicable la regla de descarte
que este proyecto ya tenía escrita. Las dos listas ahora terminan en `error,errorDescription`.
**Un recorte de campos puede borrar la evidencia de que la corrida falló**, y eso vale para
cualquier fuente que se agregue después.

**Sin estrenar, igual que el resto del kit:** no se probó dentro de ChatGPT, por no tener cuenta
paga. Las dos operaciones de la Action responden por API en tiempos y tamaños que una Action
tolera.

### Pendientes, en orden de prioridad

1. **Esperar feedback de uso real** ← acá estamos. Todo lo demás depende de eso.
2. **Regenerar el informe de ejemplo** con el formato de las decisiones 9 y 10, si el formato
   convence. Los datos siguen siendo los del 3 de septiembre y no se recalcularon: la página
   ahora declara qué le falta, pero declararlo no es lo mismo que tenerlo.
3. **Decidir si Instagram entra en la corrida automática**, que es la sexta decisión de la Fase 1
   y la segunda que más mueve el costo: ~$22/mes de datos contra ~$1,80 en X.
4. **Las decisiones abiertas de la Fase 1**, en [`REQUISITOS.md`](REQUISITOS.md). La primera es
   la que más mueve el costo: ¿vigilancia barata cada 6 h con informe completo solo
   ante un pico, o informe completo siempre?
5. **Una sola medición de ~100 menciones.** Es el único tamaño que puede distinguir 70% de 80%;
   cinco rondas de 20 no llegan.
6. **Adjudicar los casos de `tipo` que se contradicen** (ronda 4: el 15 quedó `hecho` y sus
   gemelos 16 y 18 `opinion`; el 7 quedó `ninguna` y el 4, equivalente, `perjudica`).
7. **Medir el techo humano**, como último recurso: reetiquetar los 20 de la calibración y ver
   cuánto coincide la persona consigo misma. Si ese techo es 85%, pedirle 80% al modelo es
   pedirle casi el máximo posible.

> **Reemplazar a Apify se analizó y quedó en "no ahora"**, con los motivos que sí lo
> justificarían y el plan de migración por si llega ese día:
> [`docs/alternativas-a-apify.md`](docs/alternativas-a-apify.md). El resumen: Apify es la parte
> gratis del proyecto y el reemplazo directo cuesta lo mismo por tweet sin el crédito mensual.

## El hilo de la historia, en corto

El clasificador arrancó con una sola etiqueta —positivo/neutro/negativo— y se midió tres veces:
**67%, 75%, 60%**. La precisión bajaba mientras se agregaban reglas, que fue la señal de que el
problema no estaba en las reglas.

Hubo además una medición falsa de 87% que conviene recordar: los criterios se fijaron *después*
de ver los desacuerdos, y se reetiquetaron cinco respuestas humanas correctas para que
coincidieran con un criterio del modelo que estaba mal. La persona había marcado `neutro` la
caída de DonWeb en la ronda 1, y volvió a marcarla `neutro` en la ronda 3 sin intervención: su
criterio nunca se movió.

El diagnóstico final fue que la etiqueta única obligaba a responder dos preguntas en una sola
casilla. Un nodo caído 30 horas es un hecho grave sin que nadie opine; una queja es una opinión.
El esquema nuevo las separa en `tipo` y `valencia`. Al reetiquetar los 20 de la ronda 3, la vieja
casilla `neutro` se descompuso en cuatro combinaciones distintas — incluidas cuatro menciones con
valencia desfavorable que antes eran indistinguibles del ruido.

El esquema nuevo se midió en la ronda 4 y **no arregló el sesgo**: los 6 errores de valencia van
todos en la misma dirección —el modelo carga valencia donde la persona no ve ninguna— y ninguno
invierte el signo. El sesgo nunca fue del esquema: el modelo lee cualquier texto de tono negativo
cerca de la marca como daño a la marca. Lo que sí sigue siendo cierto, cuatro rondas seguidas, es
que no se pierde ni un desfavorable.

Está todo en `docs/decisiones.md`, decisión 8, en `datos/dorado/ronda4-medicion.md`, y en el
registro al pie de `prompts/analista-sentimiento.md`.

## Protocolo de medición, para no repetir el error

1. Las menciones de cada ronda **nunca se reutilizan** entre rondas.
2. Las predicciones del modelo **se commitean antes** de que existan las etiquetas humanas.
3. Calibrar y medir van **separados**: si se ajusta el criterio mirando unos casos, la medición
   va sobre otros.
4. Ante un desacuerdo, **la etiqueta humana es la referencia**. Si un criterio del modelo la
   contradice sistemáticamente, el que está mal es el criterio.

## El kit de ChatGPT, probado contra el Actor real

Dieciséis corridas el 2026-09-04, ~$0.15 de crédito. Está en `chatgpt/` y es la vía de uso
recomendada para quien no usa Claude.

| | |
| :--- | :--- |
| 100 posteos, con `fields` | 21–22 s, 45 KB |
| 300 posteos, con `fields` | 41 s, 160 KB — el techo sensato es 150 |
| 25 posteos, sin `fields` | 30 s, 55 KB, 41 campos por posteo |
| Respuestas de un hilo (83 de 109) | 11 s |

- **`fields` no es una optimización, es lo que hace que funcione.** Sin él, 100 posteos serían
  ~200 KB de JSON crudo.
- **El tiempo no depende de la cantidad.** Lo domina el arranque del Actor: 25 posteos tardaron
  30 s y 100 tardaron 21.
- **El Actor falla en silencio.** Una corrida devolvió `[{}]` —objeto vacío, HTTP 201, 6 s— en
  vez de un error. Las instrucciones descartan toda entrada sin `id` y reintentan una vez.
- **`since:` funciona** (decisión 11) y **`Get Replies` también** (decisión 10). Los dos estaban
  anotados como pendientes de verificar y los dos quedaron resueltos.
- **La ejecución automática no se puede resolver dentro de ChatGPT:** las tareas programadas no
  pueden invocar GPTs personalizados. El requisito 4 solo sale por CLI + cron, y eso cambia el
  costo de naturaleza — el modelo deja de ser una suscripción y pasa a ser factura. Está todo en
  la actualización al pie de `REQUISITOS.md`.

Lo único que no pude probar: **la Action desde dentro de ChatGPT**, por no tener cuenta paga. La
API responde como el esquema dice, en tiempos y tamaños que una Action tolera, pero el armado del
GPT está sin estrenar. Si algo falla va a ser en el paso 4 de la guía.
