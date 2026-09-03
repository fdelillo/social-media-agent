# Fase 0 — Validación sin código

Esta fase existe para responder dos preguntas **antes** de que haya código que dependa de las
respuestas:

1. ¿El actor de X devuelve datos con los que se puede trabajar? Texto completo y no truncado,
   volumen razonable, menciones recientes.
2. ¿El prompt clasifica bien, medido contra algo?

Si la respuesta a cualquiera de las dos es "no", se descubre acá — cuando cambiar de actor o
reescribir el prompt cuesta una tarde y no un refactor.

---

## Paso 1 — Cuenta y token de Apify

1. Registrarse en [apify.com](https://apify.com). No pide tarjeta.
2. **Settings → Integrations →** copiar el API token.
3. `cp .env.example .env` y completar `APIFY_TOKEN`.

## Paso 2 — Elegir el actor de X

Ver [`apify-actor-x.md`](apify-actor-x.md) para los candidatos y el criterio. Anotar ahí el que
se elija, junto con la forma de su input y su costo por corrida — la Fase 1 lee ese archivo
para saber qué mandar.

## Paso 3 — Conectar el MCP

Seguir [`../mcp/README.md`](../mcp/README.md). Verificación: pedirle al modelo que liste las
herramientas disponibles; tiene que aparecer el actor elegido y **solo** ese.

## Paso 4 — Tres corridas de perfil distinto

Correr el actor sobre tres objetivos deliberadamente diferentes, ~100 menciones cada uno:

| Perfil | Por qué | Ejemplo |
| :--- | :--- | :--- |
| Marca grande y polarizada | Mucho volumen, sentimiento mezclado, ironía y sarcasmo | una marca con detractores activos |
| Marca chica | Poco volumen: prueba qué pasa cuando hay 8 menciones y no 100 | un negocio local o un producto de nicho |
| Persona | El lenguaje sobre personas es más ambiguo que sobre productos | una figura pública |

Guardar el JSON crudo de cada corrida en `datos/crudo/` con el nombre
`<objetivo>-<fecha>.json`. **No editarlo.**

Al terminar, revisar a ojo: ¿el texto viene completo o truncado? ¿las fechas son recientes?
¿hay retweets y spam inflando el conteo? Anotar lo que aparezca en `apify-actor-x.md`.

## Paso 5 — El set dorado

**Este es el entregable real de la fase.**

Tomar ~30 menciones mezclando las tres corridas — a propósito eligiendo los casos difíciles, no
los obvios: ironía, quejas educadas, elogios tibios, menciones donde la marca aparece de paso
sin ser el sujeto. Etiquetar **a mano**, sin mirar lo que dice el modelo, en
`datos/dorado/set-dorado.json`:

```json
[
  {
    "id": "1234567890",
    "texto": "el texto completo de la mención",
    "sentimiento": "negativo",
    "nota": "ironía: 'gran servicio' dicho en queja"
  }
]
```

El campo `nota` es opcional pero vale la pena en los casos límite: es lo que después explica
por qué el modelo se equivocó.

Treinta es poco para una estadística seria y suficiente para detectar un prompt malo. La
alternativa realista no es un set más grande — es no tener vara.

## Paso 6 — Medir

Correr [`../prompts/analista-sentimiento.md`](../prompts/analista-sentimiento.md) sobre los
textos del set dorado (sin las etiquetas), comparar y contar aciertos.

Mirar los desacuerdos uno por uno. En la práctica se concentran en dos lugares:

- **Ironía y sarcasmo**, que el modelo tiende a leer literalmente.
- **La frontera neutro/negativo**: una queja factual y sin carga emotiva ("no llegó el pedido")
  es fácil de leer como neutra, y para reputación no lo es.

Ajustar el prompt sobre esos casos — agregando ejemplos concretos a la sección de calibración —
y volver a medir. Registrar cada iteración al pie de `analista-sentimiento.md`, con el número.

## Paso 7 — Un informe completo

Con las menciones ya clasificadas de una de las corridas, correr
[`../prompts/redactor-informe.md`](../prompts/redactor-informe.md) y guardar el resultado como
`reportes/ejemplo-<marca>.md`. Ese archivo se versiona: es la referencia contra la cual se
compara la salida del CLI en la Fase 1.

---

## Criterio de salida

Se pasa a la Fase 1 cuando se cumplen las dos:

- [ ] **≥80% de coincidencia** con el set dorado, con los desacuerdos revisados y entendidos.
- [ ] Un informe de ejemplo que te resulte presentable a un tercero.

Si el actor devuelve texto truncado o volumen inservible, **no se sigue**: se cambia de actor y
se repite desde el paso 2. Ese es exactamente el descubrimiento que esta fase existe para
adelantar.
