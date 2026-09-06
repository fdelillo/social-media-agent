# Armar el agente en ChatGPT

Esta carpeta tiene todo para montar el agente como un **GPT propio** dentro de ChatGPT, sin
instalar nada y sin escribir una línea de código. Toma unos diez minutos. Cubre dos redes: **X**,
que es la fuente principal, e **Instagram**, que funciona distinto y cuesta bastante más.

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

El plan gratuito trae ~$5 de crédito por mes, que se renuevan. **En X eso es prácticamente
gratis**: un informe de 100 menciones cuesta $0,015, unos 330 informes por mes.

**En Instagram no.** Los actores de Instagram cuestan **18 veces más** ($2,70 por 1.000
resultados contra $0,15), así que un informe de Instagram sale ~$0,18 y en el mismo crédito
entran unos 18. Si vas a usar Instagram seguido, tenelo presente antes y no después.

## Paso 2 — Crear el GPT

En ChatGPT: **Explorar GPT → Crear**, y después la pestaña **Configurar**.

| Campo | Qué poner |
| :--- | :--- |
| Nombre | El que quieras, por ejemplo `Escucha` |
| Descripción | Analiza menciones en X e Instagram y devuelve un informe de reputación |
| Instrucciones | El contenido completo de [`instrucciones.md`](instrucciones.md) |

## Paso 3 — Subir los tres prompts como Conocimiento

En **Conocimiento → Subir archivos**, subir estos tres:

- [`../prompts/analista-sentimiento.md`](../prompts/analista-sentimiento.md)
- [`../prompts/redactor-informe.md`](../prompts/redactor-informe.md)
- [`../prompts/fuente-instagram.md`](../prompts/fuente-instagram.md)

**Este paso no es opcional.** Los criterios, el formato del informe y el flujo de Instagram suman
unos 22.000 caracteres, y el campo de Instrucciones admite 8.000 — hoy usa 7.985 de esos 8.000.
Por eso las instrucciones son el flujo de X y las reglas duras, y todo el detalle vive en esos
tres archivos. Si salteás el tercero, el agente va a improvisar Instagram y se va a equivocar.

En **Capacidades**, alcanza con dejar activado lo que venga por defecto. No hace falta navegación
web ni generación de imágenes.

## Paso 4 — La Action

En **Acciones → Crear nueva acción**:

1. **Esquema:** pegar el contenido de [`accion-apify.json`](accion-apify.json).
2. **Autenticación:** *Tipo de autenticación* → **Clave de API**, *Tipo de autenticación* →
   **Bearer**, y pegar el token de Apify del paso 1.
3. Guardar. Deberían aparecer **dos** operaciones: `buscarMenciones` (X) y `buscarInstagram`.

Las dos usan el mismo token de Apify, así que se configuran una sola vez.

## Paso 5 — Probar

Empezá barato, con 25 menciones:

> Analizá las menciones de **Nike**. Su cuenta oficial es @Nike. Traé 25.

Si eso anda, subí a 100 para un informe de verdad. También podés pedirle una ventana temporal
—*"las últimas 6 horas"*— o que además traiga **las respuestas que cuelgan de tus propios
posteos**, que es donde suele estar lo más accionable.

Para probar Instagram, dale el usuario de la cuenta:

> Analizá los comentarios en los posteos de **@natgeo** en Instagram.

La primera llamada es barata a propósito: confirma que el usuario existe antes de gastar.

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
| **Instagram:** dice que la cuenta no existe | Puede ser cierto. El handle de Instagram no suele ser el mismo que el de X: verificalo abriendo el perfil en el navegador. |
| **Instagram:** trae un hashtag raro con cero posteos | El Actor cayó a un fallback de Google e inventó el hashtag. El agente lo descarta; si se cuela, no lo tomes como dato. |
| **Instagram:** pediste 30 comentarios y trajo 15 | Es el techo del plan gratuito de Apify, no un error. Son 15 por posteo y 21 etiquetados por cuenta. |
| **Instagram:** los etiquetados son cuentas promocionales | Instagram no tiene filtros de calidad: no hay `min_faves` ni `Top`. El agente descarta las que etiquetan más de cinco cuentas, pero algo pasa. |
| **Instagram:** el gasto sube rápido | El límite es **por URL**: tres posteos a 15 comentarios son 45 resultados, no 15. |
| **Instagram:** vuelve `[{}]`, un objeto vacío | Alguien sacó `error,errorDescription` de la lista de `fields`. El recorte borra el motivo del fallo y deja el error irreconocible. Restaurá la lista completa del esquema. |
| **Instagram:** pediste las últimas horas y dice que no hay nada | Casi seguro es esto: el filtro de fecha mira la fecha **del posteo**, no la del comentario. Si la cuenta no posteó en esa ventana, descarta todo. Pedile los comentarios sin ventana y que filtre él. |

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

## Instagram funciona distinto, y conviene saber en qué

**En Instagram no se puede buscar por texto.** Esto no es una limitación del agente ni del plan:
Instagram no permite, a ningún precio y con ninguna herramienta, preguntar quién escribió el
nombre de una marca. Su buscador encuentra hashtags, perfiles y lugares, nunca el texto de un
posteo.

Lo que sí se puede ver son dos cosas, y son valiosas:

| Qué | De dónde sale |
| :--- | :--- |
| **Lo que te comentan** | Los comentarios en tus propios posteos. Es la fuente principal y donde está el reclamo del cliente. |
| **Lo que te etiqueta** | Los posteos de terceros donde etiquetaron a tu cuenta. |

Y una que no: **lo que se dice de vos sin nombrarte.** En X es la mayor parte del informe; en
Instagram no existe. El informe de Instagram lo dice en una línea, para que nadie crea que vio
más de lo que vio.

Tres números medidos el 5 de septiembre de 2026, en 14 corridas reales:

| | |
| :--- | :--- |
| Comentarios de un posteo | 15 como máximo en plan gratuito (el posteo declaraba 62) |
| Etiquetados de una cuenta | 21 como máximo en plan gratuito |
| Recorte de `fields` | 84 KB → 2,3 KB por tres posteos, **37 veces menos** |

Los dos techos se levantan pagando el plan Starter de Apify. El tercero es lo que hace que
Instagram entre en una conversación de ChatGPT: sin `fields`, cien posteos serían 2,8 MB.

**La advertencia sobre los etiquetados.** A diferencia de X, Instagram no ofrece ningún filtro de
calidad: no hay mínimo de likes, ni idioma, ni orden por relevancia. Los 21 que llegan son
arbitrarios y buena parte son cuentas que etiquetan decenas de marcas de golpe para que alguna
las repostee — en la prueba, la primera etiquetaba 26 a la vez. El agente descarta las que
etiquetan más de cinco, pero tomá ese bloque como una muestra sucia, no como un censo.

## Buscar solo las últimas X horas

**En X, sí, y está verificado.** Pedíselo en palabras: *"analizá las últimas 6 horas"*, y el
agente traduce eso a un filtro de fecha y hora que el Actor respeta. En la prueba, una ventana de
6 horas devolvió únicamente posteos de esas 6 horas.

**En Instagram funciona distinto, y conviene saberlo antes de creerle a un resultado vacío.** El
filtro de fecha de Instagram mira **la fecha del posteo, no la del comentario**. Si le pedís "las
últimas 6 horas" a una marca que postea una vez por semana, descarta el posteo entero y no
devuelve nada — aunque tenga doscientos comentarios de esta mañana. Medido: una ventana de 1 hora
sobre un posteo del día anterior devolvió un error, no una lista vacía.

El agente lo resuelve solo: en Instagram trae los comentarios más nuevos y los filtra por su
propia fecha. Pero tiene dos costos que te va a declarar. **Se paga lo que se descarta**, porque
acá el recorte no se puede hacer en origen como en X. Y **el techo de 15 comentarios por posteo
sigue puesto**: si en esas 6 horas hubo 200, viste 15 y no hay forma de saber cuántos faltaron.

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
| Las menciones en X | ~$0,015 por informe | ~$1,80 / mes |
| Las menciones en Instagram | ~$0,18 por informe | **~$22 / mes** |
| El análisis | incluido en tu plan | **~$42 / mes** |

O sea que la automatización convierte una herramienta gratis en una de unos cuarenta dólares al
mes — sesenta si Instagram entra en la corrida automática. En X **el gasto es del análisis y no
de los datos**; Instagram es la excepción, y por eso su lugar en un esquema automático es una
decisión aparte y no un detalle.

Antes de construir nada conviene contestar si hacen falta cuatro informes completos por día, o si
alcanza con **vigilar barato y analizar caro**: contar menciones y detectar un pico no necesita
análisis, y disparar el informe completo solo cuando algo se mueve baja el costo casi diez veces.
Con Instagram esa pregunta pesa el doble, porque ahí ni siquiera vigilar es barato.

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

**Qué red ver para qué.** X y Instagram no responden la misma pregunta y no se reemplazan. X
contesta *qué se dice de vos*, incluso entre gente que no te nombra ni te sigue. Instagram
contesta *qué te dicen a vos*, que es más chico, más caro y más accionable: son tus clientes
escribiendo debajo de tu contenido. Si tenés que elegir una, X para reputación e Instagram para
atención.

**Lo que todavía no hace bien.** La frontera entre `opinion` y `hecho` es la parte floja: la
última medición dio 65% de coincidencia con etiquetas humanas contra 70% en valencia, sobre 20
menciones. Los conteos gruesos y las citas son confiables; el reparto exacto entre las dos
columnas se lee como orden de magnitud. Está todo declarado en
[`../reportes/README.md`](../reportes/README.md).

**Lo que no puede hacer, por diseño.** No tiene memoria entre corridas: cada informe es una foto,
no una película. No puede decir "subió" ni "bajó" respecto de la corrida anterior, ni correr solo
cada X horas — necesita a alguien escribiéndole. Y en Instagram **no puede buscar por texto**,
porque eso no existe: ahí ve lo que te comentan y lo que te etiqueta, nada más.

## Un ejemplo de salida

[`../reportes/ejemplo-donweb.md`](../reportes/ejemplo-donweb.md) es un informe real sobre 100
menciones, para ver la forma antes de correr nada.
