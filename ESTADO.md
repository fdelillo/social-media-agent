# Dónde quedamos

Última sesión: **2026-09-04**. Fase 0 en curso, sin código todavía — pero el agente ya se
puede usar: montado en un cliente de chat, sin CLI.

## Lo que está hecho

| Paso | Estado |
| :--- | :--- |
| 1. Cuenta y token de Apify | ✅ `.env` con `APIFY_TOKEN`, plan FREE |
| 2. Elegir el actor de X | ✅ `scrape.badger~twitter-tweets-scraper`, $0.15/1.000, validado por API |
| 3. Conectar el MCP | ⏭️ opcional, no está en el camino crítico (se usa `curl`) |
| 4. Tres corridas | ✅ 300 menciones: MercadoLibre, DonWeb, Jorge Macri — en `datos/crudo/` |
| 5. Set dorado | ✅ 30 etiquetadas, más 4 rondas de medición |
| 6. Medir y ajustar | ⏸️ pausado a propósito: 4 rondas sin converger (ver abajo) |
| 7. Informe de ejemplo | ✅ [`reportes/ejemplo-donweb.md`](reportes/ejemplo-donweb.md), 100 menciones |

Gasto de Apify hasta ahora: ~$0.05 de los $5 mensuales.

## Lo que pasa mañana

**Pulir el informe.** Está en [`reportes/ejemplo-donweb.md`](reportes/ejemplo-donweb.md), sobre
las 100 menciones de DonWeb. Ese archivo es ahora la vara de la Fase 1: lo que genere el CLI se
compara contra él. Leerlo como si lo recibiera un tercero y anotar qué sobra, qué falta y qué no
se entiende.

**El ciclo de medición queda pausado, y es una decisión, no un olvido.** Cuatro rondas dieron
67%, 75%, 60% y (65% / 70%) sin converger, y con 20 menciones por ronda el intervalo de confianza
al 95% es de ±20 puntos: el 80% del criterio de salida cae dentro del intervalo de casi todas las
mediciones hechas. Una ronda de 20 no puede demostrar que se pasó ni que no. Seguir agregando
rondas era comprar ruido.

Lo que sí se hizo antes de pausar fue el arreglo que no dependía de ninguna decisión: el criterio
del **objetivo como escenario** ya está en la calibración del prompt. Explicaba 5 de los 6 errores
de valencia de la ronda 4.

> ⚠️ **El informe de ejemplo quedó atrás del prompt.** La decisión 9 —`relacion`, los dos modos
> y la sección *Quién te está hablando*— ya está escrita en `redactor-informe.md`, pero
> `reportes/ejemplo-donweb.md` es anterior y no la refleja. Regenerarlo es una decisión
> pendiente, a propósito: primero conviene saber si el formato nuevo convence.

### Pendientes que quedan anotados, en orden de prioridad

1. **Pulir el informe.** Incluye decidir si se regenera con el formato de la decisión 9.
2. **Una sola medición de ~100 menciones**, después del informe. Es el único tamaño que puede
   distinguir 70% de 80%; cinco rondas de 20 no llegan.
3. **Adjudicar los casos de `tipo` que se contradicen** (ronda 4: el 15 quedó `hecho` y sus
   gemelos 16 y 18 `opinion`; el 7 quedó `ninguna` y el 4, equivalente, `perjudica`).
4. **Medir el techo humano**, como último recurso: reetiquetar los 20 de la calibración y ver
   cuánto coincide la persona consigo misma. Si ese techo es 85%, pedirle 80% al modelo es
   pedirle casi el máximo posible. No viola la regla de no reutilizar menciones porque no es una
   medición del modelo.

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

Nueve corridas el 2026-09-04, ~$0.06 de crédito. Lo que se aprendió:

| | |
| :--- | :--- |
| 100 posteos, con `fields` | **21–22 s**, 45 KB, 9 campos por posteo |
| 25 posteos, sin `fields` | 30 s, 55 KB, **41 campos** por posteo |
| Rango total observado | 6 s a 31 s |

- **El techo de `max_results` era más alto de lo asumido.** Se había recomendado 50 por miedo al
  timeout; 100 tarda 21 segundos. El kit pasó a 100.
- **El tiempo no depende de la cantidad.** Lo domina el arranque del Actor: una corrida de 25
  tardó 30 s y una de 100 tardó 21 s. Pedir menos no acelera nada.
- **`fields` no es una optimización, es lo que hace que funcione.** Sin él, 100 posteos serían
  ~200 KB de JSON crudo.
- **`-from:` funciona.** Estaba documentado en `apify-actor-x.md` pero nunca se había ejecutado.
- **El Actor falla en silencio.** Una de las nueve corridas devolvió `[{}]` —un objeto vacío,
  HTTP 201, en 6 segundos— en vez de un error. No es una mención: es una corrida fallida. El
  reintento salió bien. Las instrucciones del GPT ahora descartan toda entrada sin `id` y
  reintentan una vez, porque un consumidor ingenuo lo leería como "una mención".
