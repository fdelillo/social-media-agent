# Dónde quedamos

Última sesión: **2026-09-04**. Fase 0 en curso, sin código todavía — pero el agente ya se
puede usar: montado en un cliente de chat, sin CLI.

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

Gasto de Apify hasta ahora: ~$0.20 de los $5 mensuales (300 menciones de los corpus + 16 corridas de prueba del kit).

## Lo que pasa mañana

**Nada, hasta que llegue feedback de uso real.** El kit de ChatGPT está terminado y compartido:
[`chatgpt/`](chatgpt/) más una guía de puesta en marcha publicada como página. Alguien lo va a
armar con su propio token y usarlo; lo que devuelva decide qué se toca después.

### Lo publicado, con su link

Estas páginas **no se pueden reconstruir desde el repo**: republicar el mismo archivo desde otra
conversación crea una página nueva con otra URL. Si hay que actualizarlas, se actualizan sobre
estos links.

| Qué | Link | Se genera desde |
| :--- | :--- | :--- |
| Guía de puesta en marcha en ChatGPT | https://claude.ai/code/artifact/5251dedc-ca98-4e92-9a05-88c772274d9a | `chatgpt/instrucciones.md` + `chatgpt/accion-apify.json` |
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

### Pendientes, en orden de prioridad

1. **Esperar feedback de uso real** ← acá estamos. Todo lo demás depende de eso.
2. **Regenerar el informe de ejemplo** con el formato de las decisiones 9 y 10, si el formato
   convence.
3. **Las cinco decisiones de la Fase 1**, en [`REQUISITOS.md`](REQUISITOS.md). La primera es
   nueva y es la que más mueve el costo: ¿vigilancia barata cada 6 h con informe completo solo
   ante un pico, o informe completo siempre?
4. **Una sola medición de ~100 menciones.** Es el único tamaño que puede distinguir 70% de 80%;
   cinco rondas de 20 no llegan.
5. **Adjudicar los casos de `tipo` que se contradicen** (ronda 4: el 15 quedó `hecho` y sus
   gemelos 16 y 18 `opinion`; el 7 quedó `ninguna` y el 4, equivalente, `perjudica`).
6. **Medir el techo humano**, como último recurso: reetiquetar los 20 de la calibración y ver
   cuánto coincide la persona consigo misma. Si ese techo es 85%, pedirle 80% al modelo es
   pedirle casi el máximo posible.

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
