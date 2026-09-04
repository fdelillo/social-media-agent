# Ronda 4 — la medición del esquema de dos dimensiones

**Fecha:** 2026-09-04. Predicciones selladas en `ronda4-prediccion-modelo.json` antes del
etiquetado; etiquetas humanas en `ronda4.json`. 20 menciones nuevas.

| Dimensión | Aciertos | |
| :--- | :--- | :--- |
| TIPO | 13/20 | **65%** |
| VALENCIA | 14/20 | **70%** |
| Ambas a la vez | 8/20 | **40%** |

Ninguna llega al 80% del criterio de salida. **No se pasa al paso 7.**

## La tabla completa

| # | Objetivo | TIPO modelo → humano | VALENCIA modelo → humano |
| :-- | :--- | :--- | :--- |
| 1 | DonWeb | ✓ hecho | ✓ desfavorable |
| 2 | DonWeb | ✓ hecho | ✓ desfavorable |
| 3 | DonWeb | ✓ hecho | ✓ desfavorable |
| 4 | DonWeb | ✓ opinion | ✓ desfavorable |
| 5 | DonWeb | ✗ opinion → **hecho** | ✓ desfavorable |
| 6 | DonWeb | ✗ opinion → **hecho** | ✓ desfavorable |
| 7 | DonWeb | ✓ opinion | ✗ desfavorable → **ninguna** |
| 8 | MercadoLibre | ✓ hecho | ✗ desfavorable → **ninguna** |
| 9 | MercadoLibre | ✓ opinion | ✗ desfavorable → **ninguna** |
| 10 | MercadoLibre | ✓ hecho | ✗ favorable → **ninguna** |
| 11 | MercadoLibre | ✗ opinion → **hecho** | ✗ desfavorable → **ninguna** |
| 12 | MercadoLibre | ✓ hecho | ✓ ninguna |
| 13 | MercadoLibre | ✗ hecho → **opinion** | ✓ favorable |
| 14 | MercadoLibre | ✓ opinion | ✗ desfavorable → **ninguna** |
| 15 | Jorge Macri | ✗ opinion → **hecho** | ✓ desfavorable |
| 16 | Jorge Macri | ✗ hecho → **opinion** | ✓ desfavorable |
| 17 | Jorge Macri | ✓ opinion | ✓ favorable |
| 18 | Jorge Macri | ✗ hecho → **opinion** | ✓ desfavorable |
| 19 | Jorge Macri | ✓ opinion | ✓ desfavorable |
| 20 | Jorge Macri | ✓ hecho | ✓ desfavorable |

## VALENCIA: los seis errores son el mismo error

Los seis van en una sola dirección — el modelo **carga valencia donde la persona no ve
ninguna**:

- `desfavorable` → `ninguna`: 5 casos
- `favorable` → `ninguna`: 1 caso
- `ninguna` → algo: **0 casos**
- signo invertido (favorable ↔ desfavorable): **0 casos**

El modelo nunca se equivocó de signo, y sigue sin perderse un desfavorable: el dato que
sobrevivió a las tres rondas del esquema viejo sobrevive también al esquema nuevo. Lo que
falla es el **umbral**, no la lectura.

**El rediseño de dos dimensiones no tocó el sesgo real.** Separar `tipo` de `valencia` resolvió
la colisión de preguntas —eso era cierto y sigue siéndolo— pero el sesgo nunca fue del esquema:
el modelo lee cualquier texto de tono negativo cerca de la marca como daño a la marca.

### Qué distingue los cinco casos de MercadoLibre

De las 7 menciones de MercadoLibre, la persona marcó `ninguna` en 6. Las cinco que el modelo
cargó de más comparten una forma:

| # | Texto, en corto | Quién queda mal |
| :-- | :--- | :--- |
| 8 | Estafa de un comprador a un vendedor de iPhone | el estafador |
| 9 | "Me mataron la magia... de cargar dinero sin comisión" | nadie: se cerró un agujero |
| 10 | El presidente Orsi visitó las oficinas nuevas | nadie |
| 11 | Compró un ventilador diminuto, "me río para no llorar" | el vendedor |
| 14 | Ironía sobre cómo abrevia el nombre | nadie, es una pavada |

En los cinco la marca es el **escenario**, no el sujeto: algo malo pasa *en* MercadoLibre, no
*por* MercadoLibre. El prompt ya tiene un criterio cercano —"el objetivo dentro de una discusión
sobre otra cosa"— pero está escrito para argumentos políticos, y no cubre el caso de la
plataforma donde ocurren cosas de terceros.

Es un hallazgo con consecuencia de producto, más allá del prompt: para una marca del tamaño de
MercadoLibre, la mayoría de las menciones son ruido de fondo. El informe tiene que poder decir
"de 100 menciones, 60 no dicen nada sobre vos" en vez de inflar el conteo de negativas.

## TIPO: los siete errores no tienen una dirección

- `opinion` → `hecho`: 4 casos (5, 6, 11, 15)
- `hecho` → `opinion`: 3 casos (13, 16, 18)

En los cuatro primeros el modelo vio opinión donde había un hecho con adorno: la persona
aplicó la prueba del prompt —sacar al autor y ver si queda algo verificable— y le dio prioridad
al hecho aunque el texto trajera sarcasmo (#5: "la experiencia ya está comenzando a ser
increíble"), un veredicto (#6: "Un desastre") o una consigna (#15: "Este gobierno se tiene que
ir"). El criterio implícito parece ser: **si hay un evento verificable, es `hecho`, por más
editorial que lo rodee.**

Pero los casos 16 y 18 rompen esa lectura. Los dos reportan un evento verificable —una vecina
denunció; la policía clausuró un local— y la persona los marcó `opinion`. Estructuralmente son
gemelos del 15, que marcó `hecho`. Los tres son cuentas partidarias con encuadre pesado.

**Esa contradicción no es del modelo.** El criterio de `tipo` todavía no está estable en la
cabeza de nadie, y por eso los errores van para los dos lados en vez de concentrarse. Antes de
tocar el prompt en esta dimensión hay que fijar el criterio sobre esos casos, y —por el
protocolo— fijarlo sobre estos casos significa **medir sobre otros**.

## Lo que sigue

1. **Valencia** tiene un arreglo claro y una sola causa: agregar el criterio de "la marca como
   escenario" a la calibración del prompt. Es el cambio de mayor rendimiento: 5 de los 6 errores.
2. **Tipo** no tiene arreglo hasta que se adjudiquen 15 vs. 16 y 18, y el caso 7 de valencia
   (una pregunta retórica sobre no renovar el plan, que la persona marcó `ninguna` mientras
   marcó `perjudica` el llamado a reclamar del caso 4).
3. La ronda 5 va sobre menciones nuevas, como siempre.
