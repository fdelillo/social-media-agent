# Decisiones de diseño

Registro corto de las decisiones que dan forma al proyecto, con el motivo de cada una. Si
alguna se revierte, corresponde editar acá y decir por qué.

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
