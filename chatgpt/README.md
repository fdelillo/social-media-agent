# Armar el agente en ChatGPT

Esta carpeta tiene todo para montar el agente como un **GPT propio** dentro de ChatGPT, sin
instalar nada y sin escribir una línea de código. Toma unos diez minutos.

> **Necesitás ChatGPT en un plan pago** (Plus, Pro o Team): crear GPTs y usar Actions no está
> disponible en el plan gratuito.

## Por qué lo armás vos y no te paso uno hecho

Cuando alguien comparte un GPT que llama a una API, **comparte su propia clave**: todo el que lo
use gasta del crédito de quien lo armó. Armándolo vos con tu token, cada uno paga lo suyo.

Y como el modelo que analiza es tu propia suscripción de ChatGPT, **hace falta una sola
credencial**: la de Apify, que es de donde salen las menciones.

## Paso 1 — El token de Apify

1. Crear una cuenta en [apify.com](https://apify.com). No pide tarjeta.
2. **Settings → Integrations →** copiar el API token.

El plan gratuito trae ~$5 de crédito por mes, que se renuevan. Un informe de 50 menciones cuesta
menos de un centavo, así que ese crédito da para cientos de corridas. El costo real de este
proyecto nunca fue Apify.

## Paso 2 — Crear el GPT

En ChatGPT: **Explorar GPT → Crear**, y después la pestaña **Configurar**.

| Campo | Qué poner |
| :--- | :--- |
| Nombre | El que quieras, por ejemplo `Escucha` |
| Descripción | Analiza menciones en X y devuelve un informe de reputación |
| Instrucciones | El contenido completo de [`instrucciones.md`](instrucciones.md) |

## Paso 3 — Subir los dos prompts como Conocimiento

En **Conocimiento → Subir archivos**, subir estos dos:

- [`../prompts/analista-sentimiento.md`](../prompts/analista-sentimiento.md)
- [`../prompts/redactor-informe.md`](../prompts/redactor-informe.md)

**Este paso no es opcional.** Los criterios de clasificación y el formato del informe suman unos
16.000 caracteres, y el campo de Instrucciones admite 8.000. Por eso las instrucciones son el
flujo y las reglas duras, y el detalle —los casos límite, el libro de casos, el formato exacto—
vive en esos dos archivos.

En **Capacidades**, alcanza con dejar activado lo que venga por defecto. No hace falta navegación
web ni generación de imágenes.

## Paso 4 — La Action

En **Acciones → Crear nueva acción**:

1. **Esquema:** pegar el contenido de [`accion-apify.json`](accion-apify.json).
2. **Autenticación:** *Tipo de autenticación* → **Clave de API**, *Tipo de autenticación* →
   **Bearer**, y pegar el token de Apify del paso 1.
3. Guardar. Debería aparecer una operación llamada `buscarMenciones`.

## Paso 5 — Probar

Empezá barato, con 25 menciones:

> Analizá las menciones de **Nike**. Su cuenta oficial es @Nike. Traé 25.

Si eso anda, subí a 100 para un informe de verdad. También podés pedirle una ventana temporal
—*"las últimas 6 horas"*— o que además traiga **las respuestas que cuelgan de tus propios
posteos**, que es donde suele estar lo más accionable.

---

## Si algo falla

| Síntoma | Qué pasa |
| :--- | :--- |
| Error de validación al llamar a la Action | Falta `mode: "Advanced Search"`. Es obligatorio y su default no sirve. |
| Vuelve **una sola mención vacía** | El Actor falló en silencio y devolvió `[{}]` con HTTP 201. Pedile que reintente; suele salir bien a la segunda. |
| Vuelve mucho texto y el GPT se pierde | Falta el parámetro `fields`. Sin él cada posteo trae 41 campos en vez de 9. |
| La Action tarda y corta | Poco probable con 100, pero puede pasar: el tiempo lo domina el arranque del Actor, no la cantidad. Reintentar suele alcanzar. |
| Vuelven cero menciones | Los filtros son estrictos a propósito. Bajá `min_faves`, ampliá la ventana o sacá `lang:es`. |
| En las respuestas aparecen cuentas raras con miles de likes | Publicidad inyectada por X. El agente la descarta comparando el hilo; si alguna se cuela, avisale. |
| Los resultados son spam sin engagement | Se coló `query_type: "Latest"`. Tiene que ser `"Top"`. |

### Lo que está medido

Dieciséis corridas reales contra el Actor, el 4 de septiembre de 2026:

| | |
| :--- | :--- |
| 100 posteos, con `fields` | **21–22 s**, 45 KB |
| 300 posteos, con `fields` | 41 s, **160 KB** — ya roza el corte |
| 25 posteos, sin `fields` | 30 s, 55 KB |
| Respuestas de un hilo (83 de 109) | 11 s |

Tres conclusiones que cambian cómo se usa. **El tiempo no depende de cuántos posteos pidas** —lo
domina el arranque del Actor, y una corrida de 25 tardó más que una de 100—, así que pedir menos
no hace que sea más rápido. **`fields` no es una optimización, es lo que hace que funcione**: sin
él, 100 posteos serían ~200 KB. Y **150 es el techo sensato**: a 300 la llamada tarda 41 segundos
y devuelve 160 KB, que empieza a ahogar la conversación.

## ¿Trae todas las menciones?

No, y conviene saber por qué. Hay cuatro techos, y el que aprieta casi nunca es el crédito:

| Techo | Cuál es |
| :--- | :--- |
| **Lo que pedís** | El agente trae exactamente `max_results`, ni una más. |
| **Lo que existe** | Si la ventana es corta y el objetivo chico, puede haber 2 menciones y nada más. |
| **El techo práctico** | ~150. Más arriba la llamada se pone lenta y pesada. |
| **El crédito de Apify** | ~33.000 menciones al mes. A 100 por informe son unos 330 informes. |

O sea: el agente no hace un censo de todo lo que se dijo. Trae **las más relevantes** —`Top`
ordena por engagement— hasta el número que le pidas.

## Buscar solo las últimas X horas

Sí, y está verificado. Pedíselo en palabras: *"analizá las últimas 6 horas"*, y el agente traduce
eso a un filtro de fecha y hora que el Actor respeta. En la prueba, una ventana de 6 horas
devolvió únicamente posteos de esas 6 horas.

**El cuidado que hay que tener es el volumen.** Cuanto más corta la ventana, más grande tiene que
ser el objetivo para que haya algo que analizar. Medido con la misma ventana de 6 horas: sobre
una figura pública muy comentada devolvió cientos de menciones; sobre una marca chica, **dos**.
El agente avisa cuando la ventana trae menos de diez en vez de escribir un informe sobre esa
base.

## ¿Y si lo quiero cada 6 horas, sin pedírselo?

**Este GPT no puede programarse a sí mismo.** ChatGPT tiene tareas programadas —prompts que corren
solos, hasta ~1 vez por hora— pero **no pueden usar GPTs personalizados adentro**. Sirven con apps
conectadas, no con un GPT que llama a su propia API.

Automatizarlo requiere un programa corriendo en una máquina prendida, con un temporizador.
Técnicamente no es difícil: la búsqueda ya acepta ventanas de tiempo, y si corre cada 6 horas
buscando las últimas 6, las ventanas no se pisan y no hace falta recordar nada entre corridas.

**Lo que cambia es quién paga el análisis.** Hoy el que analiza es tu suscripción de ChatGPT, ya
paga; automatizado, pasa a ser una API con factura por uso:

| | Hoy, a pedido | Automático cada 6 h |
| :--- | ---: | ---: |
| Las menciones | ~$0,015 por informe | ~$1,80 / mes |
| El análisis | incluido en tu plan | **~$42 / mes** |

O sea que la automatización convierte una herramienta gratis en una de unos cuarenta dólares al
mes, **y el gasto es del análisis, no de los datos**. Antes de construir nada conviene contestar
si hacen falta cuatro informes completos por día, o si alcanza con **vigilar barato y analizar
caro**: contar menciones y detectar un pico no necesita análisis, y disparar el informe completo
solo cuando algo se mueve baja el costo casi diez veces.

**Quién lo corre.** Cualquier ejecución automática necesita algo instalado y prendido, así que
lo natural es que corra del lado de quien armó esto y que el informe llegue ya hecho. El GPT
sigue sirviendo para consultar a demanda.

**Lo que hay que decidir**, y depende de quien lo va a leer:

- **A dónde llega.** Un informe guardado en un archivo que nadie abre no sirve. Mail, Telegram,
  Slack: el canal que se use de verdad.
- **Cada cuánto se lee.** Seis horas suena bien hasta que son cuatro documentos por día.
- **Qué merece interrumpir.** Un hecho desfavorable con alcance no es lo mismo que quince
  opiniones más: lo primero se avisa al toque, lo segundo espera al informe diario.

Las decisiones abiertas están en [`../REQUISITOS.md`](../REQUISITOS.md).

## Qué esperar, y qué no

**Lo que hace bien.** Distingue *qué opina la gente* de *qué le está pasando al objetivo*, que es
la distinción que un responsable necesita: una crítica se contesta, un problema se resuelve. Y
separa tres cosas que se responden distinto: **lo que cuelga de tus propios posteos**, lo que te
etiqueta, y lo que se dice de vos sin nombrarte. Las dos primeras son una lista de tareas.

**Lo que todavía no hace bien.** La frontera entre `opinion` y `hecho` es la parte floja: la
última medición dio 65% de coincidencia con etiquetas humanas contra 70% en valencia, sobre 20
menciones. Los conteos gruesos y las citas son confiables; el reparto exacto entre las dos
columnas se lee como orden de magnitud. Está todo declarado en
[`../reportes/README.md`](../reportes/README.md).

**Lo que no puede hacer, por diseño.** No tiene memoria entre corridas: cada informe es una foto,
no una película. No puede decir "subió" ni "bajó" respecto de la corrida anterior, ni correr solo
cada X horas — necesita a alguien escribiéndole.

## Un ejemplo de salida

[`../reportes/ejemplo-donweb.md`](../reportes/ejemplo-donweb.md) es un informe real sobre 100
menciones, para ver la forma antes de correr nada.
