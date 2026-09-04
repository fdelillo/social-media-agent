# Decisiones de diseño

Registro corto de las decisiones que dan forma al proyecto, con el motivo de cada una. Si
alguna se revierte, corresponde editar acá y decir por qué.

> Este documento cubre las decisiones **de arquitectura**. Los criterios para clasificar una
> mención —qué cuenta como opinión, qué como hecho desfavorable, qué hacer con la ironía— viven
> en [`criterios-de-clasificacion.md`](criterios-de-clasificacion.md), con el caso real que
> forzó cada uno.

---

## 1. Sin infraestructura

**Decisión:** nada de Docker, n8n, Postgres ni servicios. Apify como fuente, un LLM como
cerebro, archivos en disco como todo el almacenamiento.

**Por qué:** el proyecto hermano `../agente-clipping` ya resuelve este dominio con
infraestructura completa, y lo hace a propósito, como vehículo para aprender Docker y n8n. Un
segundo proyecto que repita ese camino no agrega nada. El valor de este está en ser el camino
corto: de un nombre a un informe, sin nada que levantar.

**Consecuencia:** cuando aparezca la necesidad de persistir entre corridas, deduplicar, mostrar
evolución en el tiempo o programar tareas, **esa necesidad no se satisface acá**. Pertenece a
`agente-clipping`. Traerla de a poco significaría reconstruir su Postgres por partes, que es
exactamente lo que esta separación evita.

---

## 2. Datos reales desde el día uno

**Decisión:** no hay capa mock. La primera corrida pega contra Apify.

**Por qué:** `agente-clipping` arranca con fuente mock porque su riesgo está en la
orquestación, no en los datos. Acá el riesgo es el inverso: lo único genuinamente incierto es
si el actor de X devuelve texto usable y si el prompt clasifica bien sobre lenguaje real —
ironía, jerga, emojis. Un mock no responde ninguna de las dos preguntas, y postergarlas es
postergar el único riesgo que importa.

**Consecuencia:** se gasta crédito de Apify desde el principio, así que cada corrida se guarda
cruda y se reprocesa desde disco (ver decisión 5).

---

## 3. Validar antes de codificar

**Decisión:** la Fase 0 no escribe código. Conecta el MCP de Apify a Claude Code, corre los
prompts a mano y mide los resultados. Recién la Fase 1 empaqueta en un CLI el prompt que ya
pasó la medición.

**Por qué:** el prompt es el corazón del producto y es lo más barato de cambiar mientras siga
siendo un archivo de texto que se edita y se vuelve a correr. Envuelto en código, cada
iteración cuesta un ciclo de desarrollo. Escribir el CLI primero sería construir andamiaje
alrededor de una hipótesis sin verificar.

**Consecuencia:** el entregable real de la Fase 0 no es "el MCP andando" sino
`datos/dorado/` — menciones etiquetadas a mano. Sin esa vara, "el reporte se ve bien" es todo
lo que se puede decir, y no alcanza.

---

## 4. Dos prompts, no uno

**Decisión:** el prompt maestro del documento de visión se parte en
`prompts/analista-sentimiento.md` (clasifica) y `prompts/redactor-informe.md` (redacta).

**Por qué:** son dos trabajos con criterios de éxito distintos. Clasificar es medible: se
compara contra el set dorado y sale un número. Redactar es cualitativo. Mezclados en un solo
prompt, un informe que se lee bien puede estar apoyado en clasificaciones malas, y no hay forma
de notarlo. Separados, cada uno se corrige por su cuenta.

**Consecuencia:** la Fase 1 necesita la separación de todos modos — la clasificación es una
llamada con salida estructurada por lotes, la redacción es una sola llamada al final.

---

## 5. Crudo a disco, siempre

**Decisión:** toda corrida de Apify guarda su JSON sin tocar en `datos/crudo/`, y ningún
reanálisis vuelve a scrapear.

**Por qué:** el crédito gratuito de Apify (~$5/mes) es el recurso escaso del proyecto, mucho más
que los tokens del LLM. Reanalizar 100 menciones guardadas cuesta centavos; volver a traerlas
cuesta crédito y no devuelve exactamente lo mismo, porque el feed se movió.

**Consecuencia:** esos JSON son además los fixtures de los tests de la Fase 1. Los tests de
parseo corren sin red y sobre datos reales, no inventados.

---

## 6. Solo X primero

**Decisión:** una sola fuente hasta que esté sólida. Instagram se enchufa después como
`fuentes/instagram.py`, detrás del mismo tipo `Mencion`.

**Por qué:** son dos actores distintos, con formatos de salida distintos y presupuestos
distintos. Los de Instagram son más frágiles y más caros porque dependen del login. Arrancar
con los dos multiplica las variables justo en la fase donde hay que aislar si el problema es el
actor, el prompt o el formato.

**Consecuencia:** el informe omite la sección de Instagram mientras esa fuente no exista. El
punto de extensión está diseñado desde ahora, pero vacío.

---

## 7. El esquema de `Mencion` se comparte con agente-clipping

**Decisión:** el tipo normalizado usa los mismos campos que la tabla `mentions` del proyecto
hermano: fuente, id externo, autor, texto, url, fecha de publicación y un campo libre de
métricas.

**Por qué:** es la única pieza donde la duplicación entre los dos proyectos es un costo real y
no una consecuencia deseada de la separación. Compartiendo el esquema, un JSON producido acá se
puede cargar allá sin traducción, y viceversa.

**Consecuencia:** las métricas van en un diccionario libre y no en columnas fijas, porque cada
red expone campos distintos y no vale la pena migrar el esquema cada vez que cambia una fuente.

---

## 8. Dos dimensiones en vez de una escala de sentimiento

**Decisión:** el clasificador devuelve `tipo` (opinion / hecho) y `valencia` (favorable /
desfavorable / ninguna) como campos independientes, en lugar de una sola etiqueta
positivo/neutro/negativo.

**Por qué:** tres rondas de medición con la etiqueta única dieron 67%, 75% y 60%. La precisión
**bajaba** a medida que se agregaban reglas para resolver los desacuerdos, que es la señal de
que el problema no estaba en las reglas.

Los desacuerdos no eran errores de lectura: en ninguno el modelo entendió mal el texto. Eran
colisiones entre dos preguntas distintas que la etiqueta única obligaba a responder juntas.
Un nodo caído 30 horas que afecta a miles de empresas es un hecho grave sin que nadie opine;
una queja de un cliente es una opinión. Ambos competían por la casilla `negativo`, y `neutro`
terminaba conteniendo tanto los hechos graves como las menciones irrelevantes. En la ronda 3,
7 de los 8 errores involucraban `neutro`.

**Cómo se llegó acá, porque el error es fácil de repetir:** en la ronda 1 la persona etiquetó
`neutro` los cinco casos de la caída de DonWeb. El modelo recomendó el criterio contrario, se
aceptó, y se reetiquetaron esas cinco respuestas para que coincidieran. La medición saltó a 87%
—y era falsa: la vara se había movido hacia las predicciones. En la ronda 3, sin intervención,
la persona volvió a etiquetar `neutro` lo mismo. Su criterio nunca se movió.

**Consecuencia:** el informe mejora, porque la distinción es la que un responsable necesita.
"Qué opina la gente" y "qué le está pasando a la marca" se responden con acciones distintas:
una crítica se contesta, un problema se resuelve. El formato del informe pasa a tener esas dos
secciones en vez de una distribución de sentimiento.

**Costo:** hay que reetiquetar. El set dorado y las rondas 2 y 3 están en el esquema viejo y no
se pueden traducir automáticamente —`neutro` es ambiguo entre "hecho desfavorable" y "sin
valencia", que es justamente el problema que motivó el cambio.

---

## 9. Tres relaciones y dos modos de informe

**Decisión:** cada mención lleva un tercer campo, `relacion`, con tres valores —`propia`,
`dirigida`, `sobre`—, calculado mecánicamente al normalizar; y el informe se escribe en uno de
dos modos, **incidente** o **estado**, elegidos por lo que traen los datos y nunca por cada
cuánto se corre.

**Por qué:** el informe de ejemplo se escribió sobre una crisis con principio y final, y esa
forma no sobrevive a un objetivo sin crisis. Corrido sobre una campaña política cada seis horas,
el formato produciría la misma narrativa y dos recomendaciones inventadas cuatro veces por día.
Un informe que suena a alarma cada vez que se corre deja de leerse.

Los tres corpus de la Fase 0 mostraron además que el conteo crudo no mide lo mismo en todos los
objetivos:

| | Autores únicos | Del propio objetivo | Top 3 autores |
| :--- | ---: | ---: | ---: |
| DonWeb | 78/100 | 2 | 11% |
| MercadoLibre | 80/100 | 11 | 17% |
| Jorge Macri | 58/100 | 10 | 23% |

Diez de las cien menciones de Jorge Macri son tuits del propio Jorge Macri: el informe estaría
contando los comunicados del cliente como menciones sobre el cliente. Y nueve son de una sola
cuenta hostil. Un titular "90% desfavorable" ahí no habla de la opinión pública, habla de cuán
activa estuvo la oposición esa mañana.

**Consecuencia:** las `propia` salen de todos los conteos y se declara cuántas se excluyeron;
todo porcentaje va acompañado de cuántos autores distintos lo sostienen; y aparece una sección
nueva, *Quién te está hablando*, con las menciones `dirigida` desfavorables ordenadas por
alcance. Esa sección es la única del informe que se traduce en una lista de tareas para hoy, y
funciona igual con incidente que sin él.

**Lo que esta decisión no resuelve, y por qué no se intenta acá:** la distinción entre un
comentario colgado de un posteo del objetivo y una mención suelta en el posteo de otro **no es
recuperable con esta fuente**. Con `query_type: "Top"` el actor devuelve publicaciones con
engagement propio, y las respuestas dentro de un hilo casi nunca lo tienen: de las 100 menciones
de Jorge Macri, las 100 son publicaciones raíz. `relacion: dirigida` es la aproximación
disponible —quién te etiquetó— y es una pregunta distinta, aunque sirva para lo mismo.

Tampoco resuelve el delta entre corridas, que es lo que una cadencia de seis horas realmente
pide. Eso exige recordar la corrida anterior, deduplicar y guardar histórico, que es
exactamente lo que vive en el proyecto hermano (ver decisión 2). Este proyecto da la foto; la
película se arma allá, cargando el JSON clasificado que acá se produce.
