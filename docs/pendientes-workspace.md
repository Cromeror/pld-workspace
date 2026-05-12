---
type: workspace-backlog
---

# Pendientes — Workspace PLD

## Mapeo flujo↔diseño en toon

**Por qué**: el flujo BE es fuente de verdad; los diseños a veces divergen y hay que detectarlo antes de implementar. Hoy no existe un índice que relacione nodos del diagrama con sus screenshots — hay que leer todo el flujo para encontrar qué diseño corresponde a qué campo.

**Para qué**: poder llegar al diseño correcto con un salto, sin procesar contexto innecesario. También permite auditar inconsistencias flujo↔diseño de forma sistemática.

**Pasos**:

**Formato elegido: [toon](https://github.com/toon-format/toon)** — Token-Oriented Object Notation. Codifica arrays uniformes en una línea por fila sin repetir claves, como CSV pero con tipos y anidamiento. Vs YAML: menos tokens para leer el índice completo; vs tabla markdown: procesable estructuralmente por la LLM sin parseo visual. Ideal para mappings tabulares donde cada fila es una referencia.

1. Definir el schema toon del mapping (columnas: `nodo`, `pagina_flujo`, `variante`, `diseno`, `nota`)
2. Por cada flujo en `docs/flujos/`, generar el mapping en una **conversación A** (LLM con acceso al diagrama + screenshots):
   - Leer cada nodo del diagrama que tenga campos de datos
   - Buscar el screenshot correspondiente en `disenos/`
   - Registrar en el bloque toon cualquier inconsistencia encontrada en `nota`
3. Guardar el mapping como bloque toon al final de `flujo.md` (después del diagrama), precedido por el comentario de convención: `<!-- jarvis:llm-index type=flow-design-mapping hide=true description="..." -->`. El comentario es auto-descriptivo para que la LLM en cualquier chat nuevo entienda qué es el bloque, para qué sirve y que Jarvis no debe renderizarlo en UI.
4. Auditar en una **conversación B** independiente (LLM sin contexto previo):
   - Darle solo el `disenos/README.md` con el mapping
   - Pedirle que navegue al nodo X y describa qué ve
   - Medir: ¿llega al archivo correcto? ¿entiende la nota de inconsistencia?
   - Si tarda más de 2 saltos o malinterpreta → iterar el schema
5. Aplicar al flujo `registro-sujeto-obligado` primero (ya tiene inconsistencias documentadas)
6. Generalizar a los demás flujos una vez validado el schema

## Limitación detectada: toon no expresa edges

**Contexto**: al mapear el step 4 (Actividad vulnerable) de `registro-sujeto-obligado`, el nodo `retorno-actividad-vulnerable` tiene dos edges salientes hacia `actividad-vulnerable` y `actividad-vulnerable-realizada-en-el-domicilio`. El toon lista ambos nodos pero no expresa que el retorno llega a los dos simultáneamente ni que deben poblarse con el mismo valor — esa semántica requiere leer el diagrama.

**Opciones a evaluar** (que Jarvis revise cuál agrega más valor con menos tokens):
1. Agregar columna `comportamiento` — texto libre para reglas no obvias del step (bifurcaciones, propagación de valores)
2. Anotar solo las edges críticas en `nota` con formato `nodo-A→nodo-B+nodo-C`
3. Mantener el toon como índice de navegación y aceptar que el comportamiento requiere leer el diagrama
