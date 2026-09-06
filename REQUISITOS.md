# Requisitos para la Fase 1

Pedidos del 2026-09-03, antes de empezar a escribir código. Están acá y no en el plan porque
**dos de ellos cambian la arquitectura** que veníamos suponiendo, y uno choca con una decisión
ya tomada.

## Lo pedido

1. **Armar el agente conectando el MCP.**
2. **Multiplataforma:** que pasárselo a alguien que usa ChatGPT no sea complejo de configurar.
3. **Ventana temporal configurable:** que busque las últimas X horas, según se le pida.
4. **Ejecución automática:** que corra cada 6 horas y entregue el informe. Deseable que la
   ventana de búsqueda coincida con la cadencia — si corre cada 6 horas, que traiga las últimas 6.

---

## Lo que esto implica: el entregable cambia de forma

El plan original era un CLI. Los requisitos 1 y 2 piden lo contrario —un agente que vive dentro
de un cliente de chat— y el 4 pide algo que **ningún cliente de chat puede hacer**: correr solo,
cada 6 horas, sin que haya una persona escribiendo el prompt.

Esa es la tensión de fondo, y no se resuelve eligiendo un bando:

| | Agente MCP en un cliente de chat | CLI headless |
| :--- | :--- | :--- |
| Se lo paso a alguien de ChatGPT | fácil | tiene que instalar algo |
| Corre solo cada 6 horas | **imposible** | natural (cron / launchd) |

**La salida es construir las dos cosas sobre un mismo núcleo**, y que el paquete exponga dos
caras:

- **Un servidor MCP propio** (no solo consumir el de Apify). Cualquier cliente compatible
  —Claude Desktop, Claude Code, ChatGPT— lo conecta con una línea de configuración, sin instalar
  ni entender nada del proyecto. Esto es lo que resuelve el requisito 2 de verdad.
- **Un CLI** sobre el mismo núcleo, que es lo que el cron invoca cada 6 horas.

Los prompts, el cliente de Apify, la normalización y el redactor son los mismos en ambos casos.
Cambia solo la puerta de entrada.

> Ojo: el requisito 1 dice "conectando el MCP" —el de Apify, como consumidor. Lo que se propone
> acá es además **publicar uno propio**. Es más trabajo que un CLI pelado, y es la única forma
> de que el requisito 2 no termine en "instalate Python".

---

## Choque con la decisión 1 (sin infraestructura)

`docs/decisiones.md` dice que las corridas programadas pertenecen a `agente-clipping`, no a este
proyecto. El requisito 4 las pide acá.

**Se puede sostener sin romper la decisión, y el requisito 4 mismo da la clave.** Si la ventana
coincide con la cadencia —cada 6 horas, las últimas 6 horas— entonces **las ventanas no se
superponen**, y por lo tanto:

- no hay menciones repetidas entre corridas,
- no hace falta deduplicar,
- no hace falta base de datos.

Cada corrida sigue siendo autocontenida. La automatización es una línea de `cron` o un
`launchd`, no infraestructura. La decisión 1 se mantiene.

**Pero hay un agujero que hay que decidir:** si una corrida falla —Apify caído, sin crédito, sin
red— esa ventana de 6 horas **se pierde para siempre**, porque nadie lleva registro de qué se
buscó. Las opciones, de menos a más:

1. Aceptarlo. Es una herramienta de reputación, no un sistema de auditoría.
2. Un archivo de marca de agua (`ultima-corrida.txt`) con el fin de la última ventana exitosa.
   La corrida siguiente busca desde ahí en vez de "hace 6 horas". Son tres líneas y no es una
   base de datos.
3. Persistir todo. Es reconstruir `agente-clipping`; queda descartado.

La 2 parece el punto justo, pero es tu decisión porque roza la frontera entre los proyectos.

---

## Sobre la ventana temporal

La búsqueda avanzada de X acepta `since:` y `until:` con fecha y hora, así que
`--horas 6` se traduce a un operador en la query. **Falta verificar que el actor los pase sin
tocarlos** — es lo primero que hay que probar cuando se retome, y cuesta una corrida de 5 ítems.

Si el actor no los respeta, el plan B es filtrar por `created_at` después de traer los datos,
pero eso desperdicia crédito: se paga por menciones que se descartan.

**Un riesgo de volumen que conviene tener presente:** con una ventana de 6 horas sobre una marca
chica puede no haber ninguna mención, y el informe quedaría vacío cuatro veces por día. Hay que
decidir qué hace el agente en ese caso — probablemente no emitir informe y decirlo, en vez de
mandar un documento vacío.

---

## Sobre el requisito 2 y el problema que ya conocemos

`docs/custom-gpt.md` documenta por qué el Custom GPT quedó diferido: **compartirlo comparte tu
token de Apify**, y con $5 de crédito unas pocas consultas ajenas lo agotan.

Publicar un servidor MCP propio no elimina ese problema, lo mueve: quien lo conecte va a
necesitar **sus propias credenciales** de Apify y del LLM. Eso es bueno —cada uno paga lo suyo—
pero significa que "no sea complejo configurar" tiene un piso: dos claves. El trabajo está en
que ese piso sea lo más bajo posible, no en eliminarlo.

---

## Actualización del 2026-09-04: dos incógnitas resueltas y una cerrada

**`since:` funciona.** El actor respeta los operadores temporales sin tocarlos, verificado con
ventanas de 24 y de 6 horas. El plan B de filtrar por `created_at` después de traer los datos
queda descartado, y con él el desperdicio de crédito. La ventana = la cadencia se sostiene: sin
superposición, sin deduplicar, sin base de datos.

**El riesgo de volumen es real y ahora tiene número.** La misma ventana de 6 horas devolvió
cientos de menciones sobre una figura pública muy comentada y **dos** sobre una marca chica.
Cuanto más corta la ventana, más grande tiene que ser el objetivo.

**El requisito 4 no se puede resolver dentro de ChatGPT.** Las tareas programadas existen y
corren hasta ~1 vez por hora, pero **no pueden invocar GPTs personalizados**. Eso cierra la
esperanza de que el requisito 2 y el 4 se resolvieran con la misma pieza: cualquier ejecución
automática necesita algo instalado y prendido, y un servidor MCP propio tampoco lo arregla,
porque también necesita un cliente corriendo.

**La salida práctica:** lo automático lo corre quien construye, no quien consume. El cron vive
en una máquina propia y el destinatario solo recibe el informe. El requisito 2 se resuelve con
el GPT a demanda, el 4 con el cron. Nadie del otro lado instala nada.

**Y el costo cambia de naturaleza, que es lo que más pesa sobre la decisión.** Hoy el modelo es
una suscripción ya pagada. En el CLI pasa a ser una API con factura:

| | Por mes, cada 6 horas |
| :--- | ---: |
| Apify (4 corridas × 100 menciones) | ~$1,80 de los $5 gratuitos |
| **El modelo** (~$0,35 × 120 corridas) | **~$42** |

Automatizar convierte una herramienta gratis en una de ~$40 mensuales, y el gasto es del modelo,
no de los datos. Eso abre una pregunta de diseño previa a escribir código: **¿vigilar barato y
analizar caro?** Contar volumen y detectar un pico no necesita LLM; disparar el informe completo
solo cuando algo se mueve bajaría el costo casi un orden de magnitud.

---

## Qué hay que decidir antes de escribir código

- [ ] ¿Vigilancia barata cada 6 h + informe completo solo ante un pico, o informe completo
      siempre? Es la decisión que más mueve el costo.
- [ ] **¿Instagram entra en la corrida automática, o queda solo a demanda?** Es nueva, del
      2026-09-05, y es la segunda que más mueve el costo: Instagram cuesta 18× lo que X ($2,70
      por 1.000 resultados contra $0,15), así que cuatro corridas diarias son ~$22/mes de datos
      contra los ~$1,80 de X. Es el único caso donde vigilar tampoco sale barato. El detalle
      está en la decisión 12 y en [`docs/apify-actor-instagram.md`](docs/apify-actor-instagram.md).

      **Y hay un agravante del 2026-09-06 que pesa sobre el requisito 4.** La cadencia de 6 horas
      se sostenía en que la ventana se acota en origen: sin superposición, sin deduplicar, sin
      base de datos. **En los comentarios de Instagram eso no se puede hacer.** El filtro de
      fecha del Actor mira la fecha del posteo, no la del comentario, así que una ventana de 6
      horas descarta el posteo padre entero y devuelve un error en vez de los comentarios
      frescos que cuelgan de él. La única salida es traer los más nuevos y descartar por
      `timestamp` — es decir, **pagar cuatro veces por día por resultados que se tiran**, a 18×
      el precio de X. La propiedad "ventana = cadencia" sigue siendo cierta en X; en Instagram
      cuesta plata cada seis horas.
- [ ] ¿Servidor MCP propio + CLI, o solo CLI con cron? (el MCP ya no resuelve el requisito 4)
- [ ] ¿Marca de agua para no perder ventanas ante una corrida fallida, o se acepta perderlas?
- [ ] ¿Qué hace el agente cuando la ventana no trae menciones?
- [ ] ¿A dónde "entrega" el informe? Archivo, mail, mensaje — no está definido y cambia el trabajo.
