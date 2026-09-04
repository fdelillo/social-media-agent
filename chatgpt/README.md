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

Si eso anda, subí a 50 para un informe de verdad.

---

## Si algo falla

| Síntoma | Qué pasa |
| :--- | :--- |
| Error de validación al llamar a la Action | Falta `mode: "Advanced Search"`. Es obligatorio y su default no sirve. |
| La Action tarda y corta | La corrida sincrónica no llegó a terminar. Bajá `max_results` a 25. Es el motivo por el que 50 es el techo recomendado. |
| Vuelve mucho texto y el GPT se pierde | Falta el parámetro `fields`. La respuesta cruda trae ~40 campos por posteo. |
| Vuelven cero menciones | Los filtros son estrictos a propósito. Bajá `min_faves`, ampliá la ventana o sacá `lang:es`. |
| Los resultados son spam sin engagement | Se coló `query_type: "Latest"`. Tiene que ser `"Top"`. |

## Qué esperar, y qué no

**Lo que hace bien.** Distingue *qué opina la gente* de *qué le está pasando al objetivo*, que es
la distinción que un responsable necesita: una crítica se contesta, un problema se resuelve. Y
separa a quien te habla de quien habla de vos, que es lo que se traduce en tareas concretas.

**Lo que todavía no hace bien.** La frontera entre `opinion` y `hecho` es la parte floja: la
última medición dio 65% de coincidencia con etiquetas humanas contra 70% en valencia, sobre 20
menciones. Los conteos gruesos y las citas son confiables; el reparto exacto entre las dos
columnas se lee como orden de magnitud. Está todo declarado en
[`../reportes/README.md`](../reportes/README.md).

**Lo que no puede hacer, por diseño.** No tiene memoria entre corridas: cada informe es una foto,
no una película. No puede decir "subió" ni "bajó", ni correr solo cada X horas. Y no ve los
comentarios que cuelgan de un posteo, solo las publicaciones con engagement propio.

## Un ejemplo de salida

[`../reportes/ejemplo-donweb.md`](../reportes/ejemplo-donweb.md) es un informe real sobre 100
menciones, para ver la forma antes de correr nada.
