# Set dorado

> Los criterios para decidir cada etiqueta están en
> [`../../docs/criterios-de-clasificacion.md`](../../docs/criterios-de-clasificacion.md).

Menciones etiquetadas **a mano**, sin mirar lo que dice el modelo. Es la única vara contra la
cual se puede afirmar que el prompt clasifica bien; sin esto, lo único que se puede decir es
que el informe se ve bien, y eso no alcanza.

Este directorio **sí se versiona** (a diferencia de `../crudo/`): es trabajo humano que no se
puede regenerar.

## Cómo armarlo

Ver [`../../docs/fase-0.md`](../../docs/fase-0.md), paso 5. En resumen: ~30 menciones mezcladas
de las tres corridas, eligiendo **a propósito los casos difíciles** — ironía, quejas educadas,
elogios tibios, menciones donde el objetivo aparece de paso. Los casos obvios no distinguen un
prompt bueno de uno malo.

## Formato

`set-dorado.json`:

```json
[
  {
    "id": "1234567890",
    "objetivo": "Nike",
    "texto": "el texto completo de la mención, tal como vino",
    "sentimiento": "negativo",
    "comentario": "ironía: elogia la espera de 45 minutos"
  }
]
```

`sentimiento` es una de tres palabras: `positivo`, `neutro` o `negativo`. **No hay puntaje ni
escala numérica** — el `score` de −1.0 a 1.0 lo produce el clasificador, pero el set dorado no
lo necesita: se mide solo la categoría.

`comentario` es texto libre y opcional. Conviene completarlo en los casos límite, porque es lo
que después explica por qué el modelo se equivocó y de dónde salen los ejemplos de calibración
que se agregan al prompt. En los casos obvios se deja vacío.

## Cómo se usa

Se le pasan al modelo los `texto` sin las etiquetas, con
[`../../prompts/analista-sentimiento.md`](../../prompts/analista-sentimiento.md), y se cuentan
los aciertos. El resultado de cada iteración se anota en la tabla al pie de ese prompt.
