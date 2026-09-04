# Dónde quedamos

Última sesión: **2026-09-04**. Fase 0 en curso, sin código todavía.

## Lo que está hecho

| Paso | Estado |
| :--- | :--- |
| 1. Cuenta y token de Apify | ✅ `.env` con `APIFY_TOKEN`, plan FREE |
| 2. Elegir el actor de X | ✅ `scrape.badger~twitter-tweets-scraper`, $0.15/1.000, validado por API |
| 3. Conectar el MCP | ⏭️ opcional, no está en el camino crítico (se usa `curl`) |
| 4. Tres corridas | ✅ 300 menciones: MercadoLibre, DonWeb, Jorge Macri — en `datos/crudo/` |
| 5. Set dorado | ✅ 30 etiquetadas, más 4 rondas de medición |
| 6. Medir y ajustar | 🔄 **acá estamos** |
| 7. Informe de ejemplo | ⬜ pendiente |

Gasto de Apify hasta ahora: ~$0.05 de los $5 mensuales.

## Lo que pasa mañana

La ronda 4 ya está medida: **tipo 65%, valencia 70%**, ninguna llega al 80% del criterio de
salida. El detalle está en [`datos/dorado/ronda4-medicion.md`](datos/dorado/ronda4-medicion.md).

Hay dos cosas pendientes, y la segunda **bloquea** a la ronda 5:

1. **Agregar a la calibración del prompt el criterio de "la marca como escenario"** — algo malo
   que pasa *en* MercadoLibre y no *por* MercadoLibre no le carga valencia. Explica 5 de los 6
   errores de valencia y no depende de que nadie decida nada.
2. **Adjudicar tres casos donde el criterio humano se contradice** (ver abajo). Sin eso, la
   dimensión `tipo` no se puede arreglar, porque sus 7 errores van para los dos lados.

Después de eso, ronda 5 sobre menciones nuevas.

### Los casos a adjudicar

- **15 vs. 16 y 18.** Los tres son cuentas partidarias que reportan un evento verificable con
  encuadre pesado. El 15 quedó `hecho` y los otros dos `opinion`. ¿Cuál es la regla?
- **7 vs. 4.** El 4 —un llamado a reclamar ante Defensa del Consumidor— quedó `perjudica`. El
  7 —"¿quién renueva plan después de este incidente?"— quedó `ninguna`. Los dos empujan clientes
  fuera de DonWeb.

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
