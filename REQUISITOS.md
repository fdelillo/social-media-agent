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

## Qué hay que decidir antes de escribir código

- [ ] ¿Servidor MCP propio + CLI, o solo CLI con cron?
- [ ] ¿Marca de agua para no perder ventanas ante una corrida fallida, o se acepta perderlas?
- [ ] ¿Qué hace el agente cuando la ventana no trae menciones?
- [ ] ¿A dónde "entrega" el informe? Archivo, mail, mensaje — no está definido y cambia el trabajo.
