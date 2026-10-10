---
name: "speckit-converge"
description: "Evaluar el código actual frente a la especificación, el plan y las tareas de la funcionalidad, y agregar a tasks.md el trabajo restante no implementado como nuevas tareas para que implement pueda completarlo."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/converge.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de la convergencia)**:

- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_converge`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con normalidad.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks sin campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según su bandera `optional`:
  - **Hook opcional** (`optional: true`):

    ```text
    ## Hooks de extensiones

    **Pre-Hook opcional**: {extension}
    Comando: `/{command}`
    Descripción: {description}

    Prompt: {prompt}
    Para ejecutar: `/{command}`
    ```

  - **Hook obligatorio** (`optional: false`):

    ```text
    ## Hooks de extensiones

    **Pre-Hook automático**: {extension}
    Ejecutando: `/{command}`
    EXECUTE_COMMAND: {command}

    Espera el resultado del comando del hook antes de continuar con el Objetivo.
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.

- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.

## Objetivo

Cerrar la brecha entre lo que exigen la especificación, el plan y las tareas de una funcionalidad y lo que implementa actualmente el código. Lee `spec.md`, `plan.md` y `tasks.md` como la **única fuente de intención** (con la constitución como restricciones de gobierno), evalúa el estado actual del código, determina qué requisitos, criterios de aceptación, decisiones del plan y tareas existentes no se cumplen, están incompletos o solo parcialmente satisfechos y **agrega cada parte del trabajo restante como una nueva tarea trazable** al final de `tasks.md`, de modo que `/speckit-implement` pueda completarla. Este comando **DEBE** ejecutarse únicamente después de que `/speckit-implement` se haya ejecutado sobre el `tasks.md` actual y después de que `/speckit-tasks` haya generado un `tasks.md` completo.

Esto **no** es una herramienta de diff y **no** realiza seguimiento de cambios. Evalúa el estado actual del código respecto de los artefactos de la funcionalidad: sin Git, sin comparación de ramas, sin historial.

## Restricciones operativas

**SOLO AGREGAR, NUNCA REESCRIBIR**: La **única** escritura permitida para este comando es agregar una nueva sección `## Phase N: Convergence` a `tasks.md`. **NO DEBE**:

- modificar `spec.md` ni `plan.md` de ninguna manera;
- reescribir, renumerar, reordenar ni eliminar ninguna tarea existente (incluidas las tareas de una fase Convergence previa);
- modificar, crear ni eliminar código de aplicación: completar las tareas agregadas corresponde a `/speckit-implement`.

Cuando el código ya satisface todo, el comando **DEBE** dejar `tasks.md` **idéntico byte por byte** (sin encabezado Convergence vacío) e informar un resultado limpio.

**Autoridad de la constitución**: La constitución del proyecto (`.specify/memory/constitution.md`) es **no negociable**. El código que viola un principio MUST constituye el hallazgo de mayor severidad y genera una tarea de corrección correspondiente. Si la constitución es una plantilla sin completar, omite correctamente las comprobaciones de constitución en lugar de fallar.

## Pasos de ejecución

### 1. Inicializar el contexto de convergencia

Ejecuta `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireSpec -RequireTasks -IncludeTasks` una sola vez desde la raíz del repositorio y analiza el JSON para obtener FEATURE_DIR y AVAILABLE_DOCS. Deriva rutas absolutas:

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md
- CONSTITUTION = `.specify/memory/constitution.md` (si existe)

Si falta `spec.md`, `plan.md` o `tasks.md`, **DETENTE** con un mensaje claro y accionable que indique el comando prerrequisito que se debe ejecutar (`/speckit-specify` si falta la especificación, `/speckit-plan` si falta el plan, `/speckit-tasks` si faltan las tareas). No produzcas salida parcial.
Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

### 2. Cargar artefactos (divulgación progresiva)

Carga únicamente el contexto mínimo necesario de cada artefacto:

**De spec.md:**

- Requisitos funcionales (FR-###).
- Criterios de éxito (SC-###): incluye únicamente elementos que requieran trabajo implementable; excluye métricas posteriores al lanzamiento y KPI de negocio.
- Historias de usuario y sus escenarios de aceptación.
- Casos límite (si existen).

**De plan.md:**

- Elecciones de arquitectura/stack y decisiones técnicas.
- Referencias al modelo de datos.
- Fases y puntos de contacto nombrados (archivos/componentes que el plan indique que se crearán o editarán).
- Restricciones técnicas.

**De tasks.md:**

- IDs de tareas (para calcular el siguiente ID y el siguiente número de fase).
- Descripciones, agrupación por fases y rutas de archivos referenciadas.

**De la constitución (si no es una plantilla sin completar):**

- Nombres de principios y declaraciones normativas MUST/SHOULD.

### 3. Construir el inventario de intención

Crea un modelo interno (no reproduzcas los artefactos sin procesar):

- **Inventario de requisitos**: una clave estable por cada FR-### / SC-### / escenario de aceptación de historia de usuario (por ejemplo, `US1/AC2`), además de las decisiones del plan y principios de la constitución que impongan obligaciones implementables.
- **Mapa de alcance del código**: a partir de las rutas de archivo nombradas en `plan.md` y `tasks.md`, además de una búsqueda por palabras clave de los conceptos descritos por cada requisito, deriva el conjunto de archivos fuente y componentes dentro del alcance de la evaluación. Limita la evaluación a estos; **no** infieras alcance más allá de lo definido por los artefactos.

### 4. Evaluar el código y clasificar hallazgos

Incluye todas las tareas existentes en el inventario de intención, independientemente del estado de su checkbox o de que pertenezcan a una fase Convergence: una afirmación de finalización no constituye evidencia. Verifica el comportamiento actual contra la especificación, el plan, las tareas y la constitución; para cadenas de tareas correctivas, evalúa el comportamiento resultante, no detalles de implementación ya sustituidos. Comprueba tanto obligaciones incumplidas como implementaciones que contradigan, excedan o queden fuera de la intención declarada.

Para cada elemento del inventario de intención, inspecciona el código actual dentro del alcance y genera un `Finding` solo cuando exista una brecha. Clasifica cada hallazgo por **tipo de brecha**:

- **`missing`**: el trabajo requerido está completamente ausente del código.
- **`partial`**: el trabajo existe, pero todavía no satisface por completo el requisito / criterio de aceptación / decisión del plan.
- **`contradicts`**: el código hace algo que entra en conflicto con la intención declarada o con un principio MUST de la constitución.
- **`unrequested`**: el código contiene trabajo no solicitado por la especificación, el plan o las tareas (se muestra para conocimiento; converge **no** elimina código, únicamente agrega una tarea para revisarlo/justificarlo o eliminarlo).

Cada `Finding` registra: un id estable, la `source-ref` a la que se vincula, el `gap-type`, una severidad y una descripción breve y legible con la evidencia (archivo/área observada).

**Casos límite:**

- **Poco o ningún código todavía**: considera todo el alcance especificado como trabajo restante `missing` en lugar de fallar.
- **No queda nada**: genera cero hallazgos y sigue la rama converged del Paso 7.

### 5. Asignar severidad

- **CRITICAL**: viola un principio MUST de la constitución o existe una brecha `missing`/`contradicts` que bloquea la funcionalidad base de una historia de usuario P1.
- **HIGH**: brecha `missing` o `partial` en un requisito funcional principal o criterio de aceptación.
- **MEDIUM**: brecha `partial` en un requisito secundario o adición `unrequested` cuya justificación no está clara.
- **LOW**: brechas parciales menores, pulido o adiciones `unrequested` de bajo riesgo.

### 6. Presentar el resumen de hallazgos en la sesión

Antes de agregar nada, muestra un resumen compacto clasificado por severidad (todavía sin escribir archivos):

## Hallazgos de convergencia

| ID | Tipo de brecha | Severidad | Fuente | Evidencia | Trabajo restante |
|----|----------------|-----------|--------|-----------|-------------------|
| F1 | missing | HIGH | FR-008 | Ejemplo: no se detectó protección append-only en path/to/module.py al escribir tasks.md | Agregar enforcement append-only |

**Métricas de resumen:**

- Requisitos / criterios de aceptación comprobados.
- Decisiones del plan comprobadas.
- Principios de la constitución comprobados (o "omitido — plantilla").
- Hallazgos por tipo de brecha (missing / partial / contradicts / unrequested).
- Hallazgos por severidad.

### 7. Agregar tareas de convergencia (o informar converged)

**Si hay uno o más hallazgos accionables** (resultado `tasks_appended`):

Agrega al **final** de `tasks.md`, conforme al contrato append-only:

1. Recorre todos los IDs de tareas existentes; deja que `M` sea el máximo. Determina el siguiente número de fase `N` (la fase existente más alta + 1).
2. Escribe un único encabezado nuevo de sección `## Phase N: Convergence`.
3. Genera un elemento de checklist por cada hallazgo accionable, ordenando primero CRITICAL/HIGH y asignando IDs con ceros a la izquierda `T{M+1:03d}, T{M+2:03d}, …`:

   ```markdown
   - [ ] T042 <descripción imperativa> per <source-ref> (<gap-type>)
   ```

   `<source-ref>` vincula la tarea con su origen: por ejemplo, `FR-003`, `SC-002`, `US1/AC2`, `plan: storage decision`, `Constitution II`.

   `<gap-type>` es uno de `missing`, `partial`, `contradicts`, `unrequested`.

   Las tareas por violaciones de la constitución **DEBEN** emitirse primero y describirse como `CRITICAL`.
4. Nunca reutilices ni renumeres IDs existentes. Si ya existe una fase Convergence previa, agrega una nueva fase con numeración independiente debajo de ella; no modifiques la anterior.

**Si no hay hallazgos accionables** (resultado `converged`):

- **No** modifiques `tasks.md` en absoluto: no agregues un encabezado de fase vacío.
- Informa: **"✅ Converged — la implementación satisface la especificación, el plan y las tareas."**
- Incluye los recuentos resumidos de lo que se comprobó.

### 8. Proporcionar siguientes acciones (handoff)

- En `tasks_appended`: indica cuántas tareas se agregaron y bajo qué fase, y recomienda ejecutar `/speckit-implement` para completarlas; señala que una ejecución posterior de converge debería encontrar menos elementos pendientes o ninguno.
- En `converged`: recomienda continuar con la revisión / abrir un PR. No es necesaria otra pasada de implementación para el alcance especificado de esta funcionalidad.

### 9. Comprobar hooks de extensiones

Después de generar el resultado, comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.

- Si existe, léelo y busca entradas bajo la clave `hooks.after_converge`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con normalidad.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks sin campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Informa el resultado de convergencia (`converged` o `tasks_appended`) dentro de la sesión antes de enumerar cualquier hook, para que los usuarios puedan decidir si ejecutan comandos opcionales de seguimiento.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según su bandera `optional`:
  - **Hook opcional** (`optional: true`):

    ```text
    ## Hooks de extensiones

    **Hook opcional**: {extension}
    Comando: `/{command}`
    Descripción: {description}

    Prompt: {prompt}
    Para ejecutar: `/{command}`
    ```

  - **Hook obligatorio** (`optional: false`):

    ```text
    ## Hooks de extensiones

    **Hook automático**: {extension}
    Ejecutando: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.

- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.
