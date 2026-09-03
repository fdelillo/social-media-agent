# Criterios de clasificación

Libro de casos. Cada criterio nació de una mención concreta que no entraba en ninguna regla, se
discutió, y se resolvió. Las menciones citadas son reales y están en `datos/dorado/`.

Este documento sirve para tres cosas: etiquetar consistente si se suma otra persona, entender
por qué el prompt dice lo que dice, y no volver a discutir algo ya resuelto.

**Las decisiones de arquitectura están en [`decisiones.md`](decisiones.md).** Acá van solo los
criterios de clasificación.

---

## El esquema

Cada mención se clasifica en **dos dimensiones independientes**. Son preguntas separadas y se
responden por separado.

**TIPO — ¿alguien está opinando?**

- `opinion` — el autor juzga, valora, se emociona, recomienda o se burla.
- `hecho` — el autor informa, describe, pregunta o reporta, sin juzgar.

La prueba: si sacás al autor de la oración, ¿queda algo verificable? Si sí, es `hecho`.

**VALENCIA — ¿esto le conviene al objetivo?**

- `favorable` / `desfavorable` / `ninguna`

Acá importa el efecto, no la intención: un hecho puede ser desfavorable aunque quien lo publica
no tenga nada contra el objetivo.

Por qué dos dimensiones y no una escala de sentimiento: ver [`decisiones.md`](decisiones.md),
decisión 8. En resumen, con una sola etiqueta se midió 67%, 75% y 60% — la precisión bajaba
mientras se agregaban reglas.

---

## Casos resueltos

### 1. Una noticia favorable no es un elogio

> 🚨TENDENCIA: Gracias a la recuperación cada vez más gente carga combustible por Mercadolibre y Rappi https://t.co/nC8klbqWvP
>
> — @julitolopez

**`hecho` + `ninguna`.** La marca aparece como evidencia de un argumento económico; el sujeto de
la frase es la recuperación, no la empresa. Nadie la está evaluando.

Si el viento a favor del mercado contara como favorable, la categoría se llenaría de cosas que
nadie dijo sobre la marca.

### 2. El contenido comercial no dice nada sobre la marca

> ATENCIÓN SOLO POR 3 horas‼️‼️ hasta 19 hs‼️ Cupón de descuento para MERCADO LIBRE‼️🚨 Te descuenta $ 25.000 En compras de $ 250.000 o más Ingresá a este link y arriba del todo aparece el código: https://t.co/Equ9aGLWlO https://t.co…
>
> — @ahorraconana

**`hecho` + `ninguna`.** Expone la marca pero nadie la evalúa. En volumen —y estas cuentas
publican mucho— llenarían la categoría favorable de spam de afiliados.

Este caso y el anterior se etiquetaron al principio con criterios opuestos: la noticia como
neutra y el cupón como positivo. Detectar esa contradicción fue lo que empezó a desarmar el
esquema viejo.

### 3. La falla en curso y la falla resuelta no son lo mismo

> DonWeb, la compañía rosarina de hosting web y registro de dominios más grande de Argentina, está dowtime en su nodo Nova desde hace más de 30 hs, sin tiempo de recuperación. Ese nodo operan empresas, sistemas de facturación, bases…
>
> — @Laterminalblog

**`hecho` + `desfavorable`.** El daño está ocurriendo.

> @DonWebOficial #Fallas en servicios de #CloudServers en nodo Nova de #DonWeb. Incident Report for #StatusDonWeb (3/9/2026) 📷Monitoring: Actualmente el 100% del nodo NOVA se encuentra operativo. Tras la recuperacion no se reporta p…
>
> — @AbogadosRosarin

**`hecho` + `favorable`.** Es el parte que informa que el servicio volvió y no hubo pérdida de
datos. Que el texto nombre la caída no lo vuelve desfavorable: lo que comunica es el final del
problema.

Esta distinción costó dos rondas. La primera versión del criterio decía que reportar una falla
era negativo, sin más, y barría también las noticias de la recuperación.

### 4. Un hecho grave no es una opinión

> La Cooperativa de Servicios Eléctricos explicó que la interrupción se debe a una caída a nivel nacional de los servidores de DonWeb. Algunos trámites y gestiones online permanecen momentáneamente in... https://t.co/fsZ0bUasqv
>
> — @laopinionsp

**`hecho` + `desfavorable`.** Un tercero explica que su servicio se cayó por culpa del objetivo.
Es información verificable, con efecto claramente desfavorable, y nadie está opinando.

Este es el caso que justifica todo el rediseño: bajo el esquema de una sola etiqueta había que
elegir entre `negativo` (perdiendo que es un hecho, no una opinión) y `neutro` (perdiendo que
es grave). Ninguna de las dos era correcta.

### 5. Preguntar es informar

> @mis2centavos de la caida del nodo NOVA de Donweb sabes algo?
>
> — @b0d2ae215bb44c7

**`hecho` + `desfavorable`.** Quien pregunta pide información, no emite un juicio. La caída
igual queda registrada, porque está en la valencia.

Se consideró contarlo como `opinion` —quien pregunta por una caída está afectado y lo hace
público— y se descartó: volvía insostenible la frontera con la pregunta genuina.

Distinto es la pregunta retórica, que afirma disfrazada de pregunta: *"¿en serio cobran eso?"*
es `opinion` + `desfavorable`.

### 6. Quien ironiza, opina

> Perdonen, pero estuvimos con el sitio caído gracias a @DonWebOficial, así que ya saben si necesitan servicio de hosting... #donweb #not
>
> — @VolemosAr

**`opinion` + `desfavorable`.** El "gracias a" es sarcasmo y el `#not` lo confirma.

> Y un día, la humanidad descubrió que todas las IAs estaban hosteadas en DonWeb. https://t.co/ldwbDKUBe3
>
> — @maxifirtman

**`opinion` + `desfavorable`.** No hay ninguna palabra negativa, y sin embargo es una burla.

La ironía invierte la valencia, y es el error más frecuente y siempre en la misma dirección:
leer literalmente un elogio que era una burla. Regla operativa: ante una contradicción entre lo
que la frase dice y lo que el hecho narrado implica, **gana el hecho narrado**.

### 7. El objetivo dentro de una discusión sobre otra cosa

> ARGENTINA COMIENZA A USAR AMAZON Se aplica una tarifa fija de 5 USD y te cobran los impuestos en la compra ¿La diferencia? Que muchos productos terminan siendo mucho mas baratos que comprarlos acá Amazon $200 / Mercado Libre $1100…
>
> — @Eltomex56

Es una comparación donde el objetivo pierde: `hecho` + `desfavorable`. Pero el sujeto del
posteo es la llegada de Amazon, no MercadoLibre.

La pregunta que resuelve estos casos: **¿alguien queda mejor o peor por lo que dice este texto,
y esa persona es el objetivo?** Cuando el objetivo se usa como ejemplo, unidad de medida o
munición en un argumento cuyo sujeto es otro, la valencia suele ser `ninguna`.

Es la debilidad más persistente del clasificador: tiende a leer al objetivo como sujeto cuando
es instrumento del argumento.

### 8. La defensa explícita es favorable

> Tensión por el acuerdo electoral en la Ciudad: LLA insiste con tener un candidato propio y el PRO ratifica a Jorge Macri https://t.co/zz4G6b3Y7C Hay que defender a Jorge Macri de esta gente de Karina junto a Pilar Ramírez que son …
>
> — @RuthFraga4

**`opinion` + `favorable`.** El texto pide explícitamente defender al objetivo. Se etiquetó
`desfavorable` por error en la calibración —arrastrado por el clima de tensión que describe la
noticia— y se corrigió.

Vale como recordatorio de que **el tono general del texto no decide**: lo que decide es qué le
hace al objetivo.

### 9. Marca empleadora

> Desde hoy ya no trabajo en DonWeb. Gran equipo y en todo sentido fue una excelente experiencia laboral. Si tienen chance de trabajar ahí, no la desaprovechen. Me tomo el resto de la semana para ver cómo sigo y vuelvo a la búsqueda…
>
> — @matiasbaldanza

**`opinion` + `favorable`.** Es elogio genuino, aunque sea de la empresa como lugar de trabajo y
no del servicio. Cuenta: la reputación de una empresa incluye cómo se habla de ella como
empleadora.

### 10. Las cuentas del propio objetivo se excluyen

En las corridas del 2026-09-03, **el 11% del corpus de MercadoLibre eran posteos de
`@marcos_galperin`** y **el 10% del de Jorge Macri, de `@jorgemacri`**.

No son menciones *sobre* el objetivo: son el objetivo hablando de sí mismo. Distorsionan siempre
en la misma dirección, porque nadie publica mal de sí mismo.

**Se excluyen en la query** con `-from:<handle>`, no al clasificar. Identificar las cuentas
oficiales y voceros es trabajo manual por objetivo, y hay que hacerlo antes de la corrida. La
señal para detectarlas después del hecho es el autor más repetido del corpus.

---

## Casos abiertos

Cosas que aparecieron y todavía no tienen criterio fijado.

### Noticias corporativas de buenas nuevas

> Mercado Libre anuncia que triplicará su capacidad logística en Chile hacia fines de 2027, sumando otros 2.500 empleos https://t.co/K0ZRSEdZji
>
> — @latercera

Se etiquetó `hecho` + `favorable`. Pero por el criterio 1, una noticia donde nadie evalúa
debería ser `ninguna`. La diferencia con el caso de la recuperación económica es que acá el
sujeto **sí es la empresa**, y lo que se reporta la beneficia directamente.

Parece correcto, pero la frontera con el caso 1 no está escrita.

### Una mención de paso con un patrocinio adentro

Un posteo que promociona un video y menciona al objetivo entre varios sponsors —"chequeado por
Hostinger, Donweb, ARC"—. No habla del objetivo en absoluto, pero lo asocia a algo bien valorado.
Sin criterio.

### El humor que no ataca

> Una rata interrumpió una rueda de prensa de Jorge Macri y se la comieron dos perros. https://t.co/jHnCdNVOMr
>
> — @somoscorta

¿Es burla al objetivo o es una anécdota graciosa que lo tiene de fondo? Se etiquetó `ninguna`,
pero no está claro que sea generalizable.

---

## Historial de mediciones

| Ronda | Esquema | Resultado | Validez |
| :--- | :--- | :--- | :--- |
| 1 | etiqueta única | 20/30 (67%) | válida |
| 1 bis | etiqueta única | 26/30 (87%) | **inválida** |
| 2 | etiqueta única | 15/20 (75%) | válida, predicciones selladas |
| 3 | etiqueta única | 12/20 (60%) | válida, predicciones selladas |
| 4 | dos dimensiones | _pendiente_ | predicciones selladas |

**Por qué la ronda 1 bis es inválida**, porque es el error más fácil de repetir: los criterios se
fijaron *después* de ver los desacuerdos, y las 6 menciones reetiquetadas se movieron todas
hacia lo que el modelo ya había predicho. Peor: en la ronda 1 la persona había marcado `neutro`
los cinco casos de la caída de DonWeb, el modelo recomendó el criterio contrario, y se
reetiquetaron esas cinco respuestas correctas para que coincidieran con un criterio equivocado.
En la ronda 3, sin intervención, la persona volvió a marcar `neutro` exactamente lo mismo.

Dato que sobrevive al cambio de esquema: **en las tres rondas el modelo acertó todos los
negativos humanos, sin excepción.** El problema nunca fue perderse una crítica, sino ver
críticas donde había información.

---

## Protocolo de medición

Salió de los errores de arriba y no es opcional:

1. Las menciones **nunca se reutilizan** entre rondas.
2. Las predicciones del modelo **se commitean antes** de que existan las etiquetas humanas.
3. Calibrar y medir van **separados**: si se ajustó el criterio mirando unos casos, la medición
   va sobre otros.
4. Ante un desacuerdo, **la etiqueta humana es la referencia**. Si un criterio del modelo la
   contradice sistemáticamente, el que está mal es el criterio.
