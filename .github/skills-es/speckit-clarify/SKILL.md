---
name: "speckit-clarify"
description: "Identificar áreas insuficientemente especificadas en la especificación actual de la funcionalidad mediante hasta 5 preguntas de aclaración muy dirigidas e incorporar las respuestas de vuelta en la especificación."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/clarify.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de la aclaración)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_clarify`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con normalidad.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks sin campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según su bandera `optional`:
  - **Hook opcional** (`optional: true`):
    ```
    ## Hooks de extensiones

    **Pre-Hook opcional**: {extension}
    Comando: `/{command}`
    Descripción: {description}

    Prompt: {prompt}
    Para ejecutar: `/{command}`
    ```
  - **Hook obligatorio** (`optional: false`):
    ```
    ## Hooks de extensiones

    **Pre-Hook automático**: {extension}
    Ejecutando: `/{command}`
    EXECUTE_COMMAND: {command}

    Espera el resultado del comando del hook antes de continuar con el Esquema.
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.
- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.

## Esquema

Objetivo: Detectar y reducir la ambigüedad o los puntos de decisión faltantes en la especificación activa de la funcionalidad y registrar las aclaraciones directamente en el archivo de especificación.

Nota: Se espera que este flujo de aclaración se ejecute (y termine) **ANTES** de invocar `/speckit-plan`. Si el usuario indica explícitamente que omite la aclaración (por ejemplo, en un spike exploratorio), puedes continuar, pero debes advertir que aumenta el riesgo de retrabajo posterior.

Pasos de ejecución:

1. Ejecuta `.specify/scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly` desde la raíz del repositorio **una sola vez** (modo combinado `--json --paths-only` / `-Json -PathsOnly`). Analiza los campos mínimos del payload JSON:
   - `FEATURE_DIR`
   - `FEATURE_SPEC`
   - (Opcionalmente captura `IMPL_PLAN`, `TASKS` para futuros flujos encadenados).
   - Si falla el análisis del JSON, aborta e indica al usuario que vuelva a ejecutar `/speckit-specify` o que verifique el entorno de la rama de la funcionalidad.
   - Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

2. **SI EXISTE**: Carga `.specify/memory/constitution.md` para obtener los principios del proyecto y las restricciones de gobierno.

3. Carga el archivo de especificación actual. Realiza un análisis estructurado de ambigüedad y cobertura usando esta taxonomía. Para cada categoría, marca el estado: Clear / Partial / Missing. Genera internamente un mapa de cobertura para priorización (no muestres el mapa sin procesar salvo que no vayas a formular preguntas).

   Alcance funcional y comportamiento:
   - Objetivos principales del usuario y criterios de éxito.
   - Declaraciones explícitas de fuera de alcance.
   - Diferenciación de roles/personas de usuario.

   Dominio y modelo de datos:
   - Entidades, atributos, relaciones.
   - Reglas de identidad y unicidad.
   - Ciclo de vida/transiciones de estado.
   - Supuestos de volumen/escala de datos.

   Interacción y flujo UX:
   - Recorridos/secuencias críticas del usuario.
   - Estados de error/vacío/carga.
   - Notas de accesibilidad o localización.

   Atributos de calidad no funcionales:
   - Rendimiento (objetivos de latencia y throughput).
   - Escalabilidad (horizontal/vertical, límites).
   - Confiabilidad y disponibilidad (uptime, expectativas de recuperación).
   - Observabilidad (logging, métricas, señales de tracing).
   - Seguridad y privacidad (authN/Z, protección de datos, supuestos de amenazas).
   - Restricciones de cumplimiento/regulatorias (si existen).

   Integración y dependencias externas:
   - Servicios/APIs externos y modos de fallo.
   - Formatos de importación/exportación de datos.
   - Supuestos de protocolo/versionado.

   Casos límite y manejo de fallos:
   - Escenarios negativos.
   - Rate limiting / throttling.
   - Resolución de conflictos (por ejemplo, ediciones concurrentes).

   Restricciones y tradeoffs:
   - Restricciones técnicas (lenguaje, almacenamiento, hosting).
   - Tradeoffs explícitos o alternativas rechazadas.

   Terminología y consistencia:
   - Términos canónicos del glosario.
   - Sinónimos evitados/términos obsoletos.

   Señales de finalización:
   - Comprobabilidad de los criterios de aceptación.
   - Indicadores medibles al estilo Definition of Done.

   Varios / marcadores:
   - Marcadores TODO / decisiones sin resolver.
   - Adjetivos ambiguos ("robusto", "intuitivo") sin cuantificación.

   Para cada categoría con estado Partial o Missing, agrega una oportunidad candidata para una pregunta salvo que:
   - La aclaración no vaya a cambiar materialmente la estrategia de implementación o validación.
   - El elemento trate específicamente del método de implementación, comparación de stack tecnológico o desglose de tareas (anótalo internamente).

4. Genera internamente una cola priorizada de preguntas candidatas de aclaración (máximo 5). **NO** las muestres todas a la vez. Aplica estas restricciones:
    - Máximo de 5 preguntas en total durante toda la sesión.
    - Cada pregunta debe poder responderse con UNA de estas formas:
       - Una selección corta de opción múltiple (2–5 opciones distintas y mutuamente excluyentes), O
       - Una respuesta de una palabra/frase corta (limita explícitamente: "Responde en <=5 palabras").
    - Incluye únicamente preguntas cuyas respuestas afecten materialmente a arquitectura, modelado de datos, descomposición de tareas, diseño de pruebas, comportamiento UX, preparación operativa o validación de cumplimiento.
    - Mantén equilibrio entre categorías: intenta cubrir primero las categorías sin resolver de mayor impacto; evita formular dos preguntas de bajo impacto cuando exista una sola área de alto impacto sin resolver (por ejemplo, postura de seguridad).
    - Excluye preguntas ya respondidas, preferencias estilísticas triviales o detalles de ejecución propios del plan (salvo que bloqueen la corrección).
    - Favorece aclaraciones que reduzcan el riesgo de retrabajo posterior o eviten pruebas de aceptación desalineadas.
    - Si quedan más de 5 categorías sin resolver, selecciona las 5 principales según la heurística (Impacto * Incertidumbre).

5. Bucle secuencial de preguntas (interactivo):
    - Presenta **EXACTAMENTE UNA** pregunta a la vez.
    - **Calidad de redacción de la pregunta (aplica a toda pregunta, de opción múltiple o respuesta corta):**
       - Empieza con `**Pregunta:**` seguido de una interrogativa completa que termine en `?`. El texto anterior al `?` debe tener sentido por sí mismo.
       - **NUNCA** uses como pregunta un simple nombre de tema, encabezado de sección o id de requisito. Por ejemplo, `Acceptance device/runtime matrix (FR-023)` es INVÁLIDO: es una etiqueta, no una pregunta.
       - Después del `?`, el único sufijo permitido es un id opcional de requisito/pregunta entre paréntesis. Formato exacto: `**Pregunta:** <interrogativa>?` o `**Pregunta:** <interrogativa>? (FR-023)`. Nunca pongas el id antes del `?`, ni uses el id (solo o junto con una etiqueta de tema) como prompt completo.
       - Inmediatamente después de la línea de la pregunta, agrega una frase en lenguaje claro de "Por qué importa" (qué está en juego para la aceptación o el release) antes de la recomendación/opciones.
       - Usa lenguaje cotidiano; introduce jerga únicamente si la defines en la misma frase. Autoevaluación: un lector que no conozca Spec Kit debe poder responder solo con la línea de Pregunta. Puede ser concisa, pero no críptica.
    - Para preguntas de opción múltiple:
       - **Analiza todas las opciones** y determina la **opción más adecuada** basándote en:
          - Mejores prácticas para el tipo de proyecto.
          - Patrones comunes en implementaciones similares.
          - Reducción de riesgos (seguridad, rendimiento, mantenibilidad).
          - Alineación con los objetivos o restricciones explícitos visibles en la especificación.
       - Presenta tu **opción recomendada de forma destacada** al inicio con razonamiento claro (1-2 frases explicando por qué es la mejor elección).
       - Formato: `**Recomendado:** Opción [X] - <razonamiento>`.
       - Después muestra todas las opciones como tabla Markdown:

       | Opción | Descripción |
       |--------|-------------|
       | A | <Descripción de la opción A> |
       | B | <Descripción de la opción B> |
       | C | <Descripción de la opción C> (agrega D/E si hace falta, hasta 5) |
       | Corta | Proporcionar una respuesta corta diferente (<=5 palabras) (inclúyela solo si una alternativa libre es apropiada) |

       - Después de la tabla, agrega: `Puedes responder con la letra de la opción (por ejemplo, "A"), aceptar la recomendación diciendo "sí" o "recomendado", o proporcionar tu propia respuesta corta.`
    - Para estilo de respuesta corta (sin opciones discretas significativas):
       - Proporciona tu **respuesta sugerida** basándote en mejores prácticas y contexto.
       - Formato: `**Sugerido:** <tu respuesta propuesta> - <razonamiento breve>`.
       - Después muestra: `Formato: Respuesta corta (<=5 palabras). Puedes aceptar la sugerencia diciendo "sí" o "sugerido", o proporcionar tu propia respuesta.`
    - Después de que el usuario responda:
       - Si el usuario responde "sí", "recomendado" o "sugerido", usa como respuesta tu recomendación/sugerencia previamente indicada.
       - En caso contrario, valida que la respuesta corresponda a una opción o cumpla la restricción de <=5 palabras.
       - Si es ambigua, solicita una desambiguación rápida (sigue contando como la misma pregunta; no avances).
       - Una vez satisfactoria, regístrala en memoria de trabajo (todavía no escribas en disco) y pasa a la siguiente pregunta en cola.
    - Deja de formular preguntas adicionales cuando:
       - Todas las ambigüedades críticas se hayan resuelto antes de tiempo (los elementos restantes de la cola dejan de ser necesarios), O
       - El usuario indique que ha terminado ("terminado", "bien", "no más"), O
       - Llegues a 5 preguntas formuladas.
    - Nunca reveles con antelación las futuras preguntas de la cola.
    - Si al inicio no existen preguntas válidas, informa inmediatamente que no hay ambigüedades críticas.

6. Integración después de CADA respuesta aceptada (enfoque de actualización incremental):
    - Mantén una representación en memoria de la especificación (cargada una sola vez al inicio) junto con el contenido sin procesar del archivo.
    - Para la primera respuesta integrada de esta sesión:
       - Asegúrate de que exista una sección `## Clarifications` (créala justo después de la sección contextual/general de más alto nivel conforme a la plantilla de la especificación, si falta).
       - Debajo, crea (si no existe) un subencabezado `### Session YYYY-MM-DD` para la fecha de hoy.
    - Agrega inmediatamente tras aceptar la respuesta una viñeta: `- Q: <pregunta> → A: <respuesta final>`.
    - Luego aplica inmediatamente la aclaración a las secciones más apropiadas:
       - Ambigüedad funcional → Actualiza o agrega una viñeta en Requisitos funcionales.
       - Interacción de usuario / distinción de actor → Actualiza Historias de usuario o la subsección Actores (si existe) con el rol, restricción o escenario aclarado.
       - Forma de datos / entidades → Actualiza Modelo de datos (agrega campos, tipos, relaciones) conservando el orden; anota las restricciones agregadas de forma concisa.
       - Restricción no funcional → Agrega/modifica criterios medibles en Criterios de éxito > Resultados medibles (convierte el adjetivo vago en una métrica u objetivo explícito).
       - Caso límite / flujo negativo → Agrega una nueva viñeta bajo Casos límite / Manejo de errores (o crea esa subsección si la plantilla incluye un marcador para ella).
       - Conflicto terminológico → Normaliza el término en toda la especificación; conserva el original solo si es necesario agregando una vez `(anteriormente denominado "X")`.
    - Si la aclaración invalida una declaración ambigua anterior, reemplázala en lugar de duplicarla; no dejes texto obsoleto contradictorio.
    - Guarda el archivo de especificación **DESPUÉS de cada integración** para minimizar el riesgo de pérdida de contexto (sobrescritura atómica).
    - Conserva el formato: no reordenes secciones no relacionadas; mantén intacta la jerarquía de encabezados.
    - Mantén cada aclaración insertada mínima y comprobable (evita deriva narrativa).

7. Validación (realizada después de CADA escritura y en una pasada final):
   - La sesión de aclaraciones contiene exactamente una viñeta por respuesta aceptada (sin duplicados).
   - Total de preguntas formuladas (aceptadas) ≤ 5.
   - Las secciones actualizadas no contienen marcadores vagos pendientes que la nueva respuesta debía resolver.
   - No queda ninguna declaración anterior contradictoria (busca y elimina alternativas ahora inválidas).
   - La estructura Markdown es válida; los únicos encabezados nuevos permitidos son: `## Clarifications`, `### Session YYYY-MM-DD`.
   - Consistencia terminológica: se usa el mismo término canónico en todas las secciones actualizadas.

8. Escribe la especificación actualizada de vuelta en `FEATURE_SPEC`.

9. **Volver a validar la checklist de calidad de la especificación** (si existe):
   - Comprueba si existe `FEATURE_DIR/checklists/requirements.md`.
   - Si **NO** existe, omite silenciosamente este paso.
   - Si existe:
     1. Lee el archivo de checklist.
     2. Identifica todas las líneas de casillas de lista de tareas de GitHub: líneas que coincidan con `- [ ]`, `- [x]` o `- [X]` (sin distinguir mayúsculas/minúsculas y tolerando espacios iniciales para elementos anidados) fuera de bloques de código. Ignora el resto del contenido (encabezados, notas, viñetas sin checkbox, metadatos).
     3. Para cada línea de checkbox, registra su estado actual (marcado o sin marcar) y el texto del elemento en una lista snapshot previa.
     4. Vuelve a evaluar cada elemento de checkbox contra la especificación **actualizada** (la versión recién guardada en el paso 7).
     5. Para cada elemento, actualiza solo si realmente cambia el estado marcado/sin marcar:
        - Si el elemento ahora cumple y estaba sin marcar: cambia `[ ]` a `[x]`.
        - Si ahora no cumple y estaba marcado: cambia `[x]`/`[X]` a `[ ]`.
        - Si el estado no cambia: deja el marcador tal cual (conserva la capitalización existente para evitar diffs cosméticos).
     6. Guarda el archivo de checklist actualizado. **Solo cambia la parte del marcador `[ ]`/`[x]` de las líneas cuyo estado haya cambiado.** Todo el resto del contenido del archivo —encabezados, metadatos, notas, orden de líneas, espacios— debe permanecer sin cambios para evitar diffs ruidosos.
     7. Compara el snapshot previo con el estado actual para calcular tres listas para el Informe de finalización:
        - **Nuevos aprobados**: elementos que cambiaron de sin marcar a marcados.
        - **Regresiones**: elementos que cambiaron de marcados a sin marcar.
        - **Aún sin marcar**: elementos que siguen sin marcar.
     8. Registra los recuentos antes/después como elementos marcados/total (por ejemplo, "12/16 → 15/16 elementos aprobados").

Reglas de comportamiento:

- Si no se encuentran ambigüedades significativas (o todas las preguntas potenciales serían de bajo impacto), responde: "No se detectaron ambigüedades críticas que justifiquen una aclaración formal." y sugiere continuar.
- Si falta el archivo de especificación, indica al usuario que ejecute primero `/speckit-specify` (**no** crees aquí una nueva especificación).
- Nunca superes 5 preguntas totales (los reintentos de aclaración para una misma pregunta no cuentan como preguntas nuevas).
- Evita preguntas especulativas sobre stack tecnológico salvo que su ausencia bloquee la claridad funcional.
- Respeta señales de terminación anticipada del usuario ("detente", "terminado", "continuar").
- Si no se formulan preguntas por cobertura completa, muestra un resumen compacto de cobertura (todas las categorías Clear) y sugiere avanzar.
- Si se alcanza la cuota con categorías de alto impacto aún sin resolver, márcalas explícitamente bajo Deferred e incluye la justificación.

Contexto para priorización: $ARGUMENTS

## Hooks obligatorios posteriores a la ejecución

**DEBES completar esta sección antes de informar al usuario que el proceso terminó.**

Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si no existe o no hay hooks registrados bajo `hooks.after_clarify`, pasa al Informe de finalización.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_clarify`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con el Informe de finalización.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks sin campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según su bandera `optional`:
  - **Hook obligatorio** (`optional: false`) — **DEBES emitir `EXECUTE_COMMAND:` para cada hook obligatorio**:
    ```
    ## Hooks de extensiones

    **Hook automático**: {extension}
    Ejecutando: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.
  - **Hook opcional** (`optional: true`):
    ```
    ## Hooks de extensiones

    **Hook opcional**: {extension}
    Comando: `/{command}`
    Descripción: {description}

    Prompt: {prompt}
    Para ejecutar: `/{command}`
    ```

## Informe de finalización

Informa la finalización (después de que termine el bucle de preguntas o haya una terminación anticipada):
- Número de preguntas formuladas y respondidas.
- Ruta de la especificación actualizada.
- Secciones modificadas (lista de nombres).
- Estado de la checklist de calidad de la especificación (si se volvió a validar `FEATURE_DIR/checklists/requirements.md`): muestra los recuentos antes/después (por ejemplo, "Spec Quality Checklist: 12/16 → 15/16 elementos aprobados") y enumera cualquier elemento cuyo estado haya cambiado, tanto nuevos marcados (sin marcar → marcado) como regresiones (marcado → sin marcar). Si queda algún elemento sin marcar, enuméralo como área que requiere atención.
- Tabla de resumen de cobertura que enumere cada categoría de la taxonomía con Estado: Resolved (era Partial/Missing y se resolvió), Deferred (supera la cuota de preguntas o el elemento restante es específicamente método de implementación, comparación de stack tecnológico o desglose de tareas), Clear (ya era suficiente), Outstanding (sigue Partial/Missing pero de bajo impacto).
- Si quedan elementos Outstanding o Deferred, recomienda si conviene continuar a `/speckit-plan` o volver a ejecutar `/speckit-clarify` más adelante, después del plan.
- Siguiente comando sugerido.

## Finalizado cuando

- [ ] Se identificaron las ambigüedades de la especificación y las aclaraciones se integraron en el archivo de especificación.
- [ ] La checklist de calidad de la especificación se volvió a validar contra la especificación actualizada (si existe `FEATURE_DIR/checklists/requirements.md`).
- [ ] Los hooks de extensiones se despacharon u omitieron conforme a las reglas de la sección Hooks obligatorios posteriores a la ejecución.
- [ ] Se informó al usuario la finalización con preguntas respondidas, secciones modificadas, estado de la checklist y resumen de cobertura.
