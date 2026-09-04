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

## Paso 2 — Elegir el actor de X ✅

**Hecho el 2026-09-03.** El elegido es `scrape.badger~twitter-tweets-scraper`: $0.15 por 1.000
menciones en plan gratuito, sin start fee, y verificado que corre por API. Su input, su salida y
los dos candidatos descartados están documentados en [`apify-actor-x.md`](apify-actor-x.md).

## Paso 3 — Conectar el MCP *(opcional)*

El MCP es una comodidad, no un requisito: sirve para pedirle corridas al modelo en lenguaje
natural. Su guía está en [`../mcp/README.md`](../mcp/README.md).

**No está en el camino crítico, y conviene que no lo esté.** El CLI de la Fase 1 va a usar la
API REST, así que probar por `curl` verifica exactamente lo que la Fase 1 va a hacer; el MCP es
otra vía de acceso y podría comportarse distinto. Todo lo que sigue usa `curl` directo, y con
eso alcanza para terminar la fase.

## Paso 4 — Tres corridas de perfil distinto

Correr el actor sobre tres objetivos deliberadamente diferentes, ~100 menciones cada uno:

| Perfil | Por qué | Ejemplo |
| :--- | :--- | :--- |
| Marca grande y polarizada | Mucho volumen, sentimiento mezclado, ironía y sarcasmo | una marca con detractores activos |
| Marca chica | Poco volumen: prueba qué pasa cuando hay 8 menciones y no 100 | un negocio local o un producto de nicho |
| Persona | El lenguaje sobre personas es más ambiguo que sobre productos | una figura pública |

La llamada, con el actor ya elegido:

```bash
set -a; . ./.env; set +a
curl -s -X POST "https://api.apify.com/v2/acts/scrape.badger~twitter-tweets-scraper/run-sync-get-dataset-items?maxTotalChargeUsd=0.10" \
  -H "Authorization: Bearer $APIFY_TOKEN" -H "Content-Type: application/json" \
  -d '{"mode":"Advanced Search","query":"<MARCA> -is:retweet lang:es min_faves:5","query_type":"Top","max_results":100}' \
  -o "datos/crudo/<marca>-$(date +%F).json"
```

Los cuatro parámetros del input no son negociables y cada uno tiene su motivo, explicado en
[`apify-actor-x.md`](apify-actor-x.md): `mode` es obligatorio y su default no sirve;
`query_type: "Top"` en vez de `"Latest"` es la diferencia entre datos usables y spam;
`-is:retweet` evita que un mismo mensaje pese diez veces; `min_faves:5` filtra cuentas nuevas
de spam; y `maxTotalChargeUsd` es el único freno real si algo sale mal.

**No editar el JSON crudo.** Al terminar, revisar a ojo: ¿el texto de `full_text` viene
completo? ¿las fechas son recientes? ¿cuántos resultados son ruido? Anotar lo que aparezca en
`apify-actor-x.md`.

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

## Paso 7 — Un informe completo ✅

**Hecho el 2026-09-04.** El informe de referencia es
[`../reportes/ejemplo-donweb.md`](../reportes/ejemplo-donweb.md), sobre las 100 menciones de
DonWeb: la caída del nodo NOVA es el caso más rico de los tres corpus, porque tiene hechos
desfavorables graves y opiniones desfavorables a la vez, que es justo lo que el esquema de dos
dimensiones existe para separar.

Se generó en dos pasos, como en la Fase 1: primero
[`../prompts/analista-sentimiento.md`](../prompts/analista-sentimiento.md) sobre las menciones
normalizadas —resultado en `datos/clasificado/donweb-2026-09-03.json`— y después
[`../prompts/redactor-informe.md`](../prompts/redactor-informe.md) sobre ese JSON.

**Se hizo antes de cerrar el paso 6, a propósito.** La medición no converge (ver abajo) y el
informe responde una pregunta que la medición no responde: si esto le sirve a alguien. Las
limitaciones de la clasificación están declaradas al frente, en
[`../reportes/README.md`](../reportes/README.md), en vez de escondidas.

---

## Criterio de salida

Se pasa a la Fase 1 cuando se cumplen las dos:

- [ ] **≥80% de coincidencia** con el set dorado, con los desacuerdos revisados y entendidos.
- [x] Un informe de ejemplo que te resulte presentable a un tercero.

### El primer criterio, revisado

Tal como estaba escrito, ese 80% **no era medible con el procedimiento que usábamos**. Cuatro
rondas de 20 menciones dieron 67%, 75%, 60% y (65% en `tipo` / 70% en `valencia`), y con n=20 el
intervalo de confianza al 95% ronda los ±20 puntos: el 80% cae dentro del intervalo de casi todas
esas mediciones. Ninguna ronda podía demostrar que se pasó, y ninguna podía demostrar que no.
Peor: la diferencia entre 60% y 75% tampoco es distinguible del ruido, así que la frase "la
precisión bajaba a medida que se agregaban reglas" hay que leerla con pinzas — el argumento que
sostiene la decisión 8 es la *forma* de los errores, no esa tendencia numérica.

**Corrección del procedimiento:** la coincidencia se mide **una sola vez sobre ~100 menciones**,
que es el tamaño mínimo que distingue 70% de 80%, y no en rondas sucesivas de 20. Las rondas
chicas siguen sirviendo para *encontrar* criterios que faltan —para eso funcionaron bien— pero no
para decidir si se pasa de fase.

Si el actor devuelve texto truncado o volumen inservible, **no se sigue**: se cambia de actor y
se repite desde el paso 2. Ese es exactamente el descubrimiento que esta fase existe para
adelantar.
