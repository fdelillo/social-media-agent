# Informes de ejemplo

Acá vive la salida de referencia del sistema: el informe que la Fase 1 tiene que poder
reproducir desde el CLI. Se versiona a propósito — es la vara contra la cual se compara lo que
genere el código.

## Qué hay

| Archivo | Objetivo | Menciones | Fuente |
| :--- | :--- | :--- | :--- |
| [`ejemplo-donweb.md`](ejemplo-donweb.md) | DonWeb | 100 | `datos/crudo/donweb-2026-09-03.json` |

## Cómo se generó

1. Las 100 menciones crudas se normalizaron al tipo compartido (id, autor, texto, url, fecha,
   métricas en un campo libre).
2. Se clasificaron con [`../prompts/analista-sentimiento.md`](../prompts/analista-sentimiento.md)
   en las dos dimensiones, `tipo` y `valencia`. El resultado quedó en
   `datos/clasificado/donweb-2026-09-03.json`, que es la entrada real del informe.
3. Se redactó con [`../prompts/redactor-informe.md`](../prompts/redactor-informe.md) sobre ese
   JSON. Todos los conteos y los alcances del informe salen de ahí y son verificables.

## Nota de método: qué vale y qué no de este informe

**La clasificación de las 100 menciones es del modelo, no está verificada a mano.** Catorce de
ellas sí tienen etiqueta humana —vienen de la ronda 4 y de la calibración del set dorado— y se
respetaron tal cual. Las otras 86 son juicio del modelo con los criterios vigentes.

Eso importa porque la última medición conocida, la ronda 4, dio **65% en `tipo` y 70% en
`valencia`** sobre 20 menciones, con un intervalo de confianza tan ancho que no distingue 70%
de 80%. Traducido a este informe:

- **Los conteos gruesos son sólidos.** El sesgo medido del modelo es cargar valencia de más
  sobre menciones que no la tienen, nunca invertir el signo ni perderse una desfavorable. Con
  90% de menciones desfavorables, unas pocas mal cargadas no mueven la conclusión.
- **El reparto entre `opinion` y `hecho` es la parte floja.** Es la frontera donde el criterio
  todavía no está estable —ni siquiera entre humanos, ver
  [`../datos/dorado/ronda4-medicion.md`](../datos/dorado/ronda4-medicion.md)— así que los
  números de esa columna se leen como orden de magnitud, no como medición.
- **Las citas textuales, los alcances y las fechas son exactos.** Salen del JSON crudo sin
  intermediación del modelo, y son la parte del informe sobre la que se puede decidir algo.

El informe se escribió igual, con esta advertencia adelante, porque la pregunta que importa no
es si el clasificador llega al 80% sino si el informe le sirve a alguien que tiene que decidir.
Esa pregunta se responde leyéndolo.
