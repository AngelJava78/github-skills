---
name: "speckit-analyze"
description: "Realizar un análisis no destructivo de consistencia y calidad entre artefactos de spec.md, plan.md y tasks.md después de generar las tareas."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/analyze.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes del análisis)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_analyze`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con normalidad.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks que no tengan un campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según el valor de su bandera `optional`:
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

    Espera el resultado del comando del hook antes de continuar con el Objetivo.
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.
- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.

## Objetivo

Identificar inconsistencias, duplicaciones, ambigüedades y elementos insuficientemente especificados entre los tres artefactos principales (`spec.md`, `plan.md`, `tasks.md`) antes de la implementación. Este comando **DEBE** ejecutarse únicamente después de que `/speckit-tasks` haya generado correctamente un `tasks.md` completo.

## Restricciones operativas

**ESTRICTAMENTE SOLO LECTURA**: **No** modifiques ningún archivo. Genera un informe de análisis estructurado. Ofrece un plan de corrección opcional (el usuario debe aprobarlo explícitamente antes de que se invoquen manualmente comandos posteriores de edición).

**Autoridad de la constitución**: La constitución del proyecto (`.specify/memory/constitution.md`) es **no negociable** dentro del alcance de este análisis. Los conflictos con la constitución son automáticamente CRÍTICOS y requieren ajustar la especificación, el plan o las tareas; no diluir, reinterpretar ni ignorar silenciosamente el principio. Si un principio necesita cambiar, debe hacerse mediante una actualización separada y explícita de la constitución, fuera de `/speckit-analyze`.

## Pasos de ejecución

### 1. Inicializar el contexto del análisis

Ejecuta `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireSpec -RequireTasks -IncludeTasks` una sola vez desde la raíz del repositorio y analiza el JSON para obtener FEATURE_DIR y AVAILABLE_DOCS. Deriva las rutas absolutas:

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md

Aborta con un mensaje de error si falta algún archivo requerido (indica al usuario que ejecute el comando de prerrequisito faltante).
Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

### 2. Cargar artefactos (divulgación progresiva)

Carga únicamente el contexto mínimo necesario de cada artefacto:

**De spec.md:**

- Descripción general/contexto
- Requisitos funcionales
- Criterios de éxito (resultados medibles; por ejemplo, rendimiento, seguridad, disponibilidad, éxito del usuario, impacto de negocio)
- Historias de usuario
- Casos límite (si existen)

**De plan.md:**

- Elecciones de arquitectura/stack
- Referencias al modelo de datos
- Fases
- Restricciones técnicas

**De tasks.md:**

- IDs de tareas
- Descripciones
- Agrupación por fases
- Marcadores de paralelismo [P]
- Rutas de archivos referenciadas

**De la constitución:**

- Carga `.specify/memory/constitution.md` para validar los principios.

### 3. Construir modelos semánticos

Crea representaciones internas (no incluyas los artefactos sin procesar en la salida):

- **Inventario de requisitos**: Para cada Requisito Funcional (FR-###) y Criterio de Éxito (SC-###), registra una clave estable. Usa el identificador explícito FR-/SC- como clave principal cuando exista y, opcionalmente, deriva también un slug de frase imperativa para facilitar la lectura (por ejemplo, "User can upload file" → `user-can-upload-file`). Incluye únicamente los Criterios de Éxito que requieran trabajo implementable (por ejemplo, infraestructura de pruebas de carga o herramientas de auditoría de seguridad) y excluye métricas posteriores al lanzamiento y KPI de negocio (por ejemplo, "Reduce support tickets by 50%").
- **Inventario de historias de usuario/acciones**: Acciones discretas del usuario con criterios de aceptación.
- **Mapa de cobertura de tareas**: Asocia cada tarea con uno o más requisitos o historias (por inferencia mediante palabras clave o patrones de referencia explícita como IDs o frases clave).
- **Conjunto de reglas de la constitución**: Extrae nombres de principios y declaraciones normativas MUST/SHOULD.

### 4. Pasadas de detección (análisis eficiente en tokens)

Concéntrate en hallazgos de alto valor. Limita a 50 hallazgos en total; agrega el resto en un resumen de excedentes.

#### A. Detección de duplicados

- Identifica requisitos casi duplicados.
- Marca la redacción de menor calidad para consolidarla.

#### B. Detección de ambigüedad

- Marca adjetivos vagos (rápido, escalable, seguro, intuitivo, robusto) que no tengan criterios medibles.
- Marca marcadores de posición sin resolver (TODO, TKTK, ???, `<placeholder>`, etc.).

#### C. Especificación insuficiente

- Requisitos con verbos pero sin objeto o resultado medible.
- Historias de usuario sin alineación con criterios de aceptación.
- Tareas que hagan referencia a archivos o componentes no definidos en la especificación/plan.

#### D. Alineación con la constitución

- Cualquier requisito o elemento del plan que entre en conflicto con un principio MUST.
- Secciones obligatorias o quality gates exigidos por la constitución que falten.

#### E. Brechas de cobertura

- Requisitos sin tareas asociadas.
- Tareas que no estén asociadas a ningún requisito/historia.
- Criterios de Éxito que requieran trabajo implementable (rendimiento, seguridad, disponibilidad) y no estén reflejados en las tareas.

#### F. Inconsistencia

- Deriva terminológica (el mismo concepto recibe nombres distintos entre archivos).
- Entidades de datos referenciadas en el plan pero ausentes en la especificación (o viceversa).
- Contradicciones en el orden de las tareas (por ejemplo, tareas de integración antes que tareas fundamentales sin nota de dependencia).
- Requisitos contradictorios (por ejemplo, uno exige Next.js y otro especifica Vue).

### 5. Asignar severidad

Usa esta heurística para priorizar hallazgos:

- **CRITICAL**: Viola un MUST de la constitución, falta un artefacto principal de especificación o existe un requisito sin cobertura que bloquea la funcionalidad base.
- **HIGH**: Requisito duplicado o contradictorio, atributo ambiguo de seguridad/rendimiento, criterio de aceptación no comprobable.
- **MEDIUM**: Deriva terminológica, falta de cobertura de tareas no funcionales, caso límite insuficientemente especificado.
- **LOW**: Mejoras de estilo/redacción, redundancia menor que no afecta el orden de ejecución.

### 6. Generar un informe de análisis compacto

Genera un informe Markdown (sin escribir archivos) con la siguiente estructura:

## Informe de análisis de la especificación

| ID | Categoría | Severidad | Ubicación(es) | Resumen | Recomendación |
|----|-----------|-----------|---------------|---------|---------------|
| A1 | Duplicación | HIGH | spec.md:L120-134 | Dos requisitos similares ... | Fusionar la redacción; conservar la versión más clara |

(Añade una fila por hallazgo; genera IDs estables con prefijo basado en la inicial de la categoría.)

**Tabla de resumen de cobertura:**

| Clave del requisito | ¿Tiene tarea? | IDs de tareas | Notas |
|---------------------|---------------|---------------|-------|

**Problemas de alineación con la constitución:** (si existen)

**Tareas sin asociación:** (si existen)

**Métricas:**

- Total de requisitos
- Total de tareas
- % de cobertura (requisitos con >=1 tarea)
- Cantidad de ambigüedades
- Cantidad de duplicados
- Cantidad de problemas críticos

### 7. Proporcionar siguientes acciones

Al final del informe, genera un bloque conciso de Siguientes acciones:

- Si existen problemas CRITICAL: recomienda resolverlos antes de `/speckit-implement`.
- Si solo hay LOW/MEDIUM: el usuario puede continuar, pero proporciona sugerencias de mejora.
- Proporciona sugerencias explícitas de comandos; por ejemplo, "Ejecuta /speckit-specify para refinar", "Ejecuta /speckit-plan para ajustar la arquitectura", "Edita manualmente tasks.md para agregar cobertura de 'performance-metrics'".

### 8. Ofrecer correcciones

Pregunta al usuario: "¿Quieres que sugiera ediciones concretas para corregir los N problemas principales?" (**No** las apliques automáticamente).

### 9. Comprobar hooks de extensiones

Después de generar el informe, comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_analyze`.
- Si el YAML no puede analizarse o es inválido, no lo omitas silenciosamente: informa al usuario que `.specify/extensions.yml` no pudo leerse (incluye el error del parser) y que no se comprobaron hooks, incluidos los hooks obligatorios (`optional: false`) registrados allí; después continúa con normalidad.
- Filtra los hooks donde `enabled` sea explícitamente `false`. Considera habilitados por defecto los hooks que no tengan un campo `enabled`.
- Para cada hook restante, **no** intentes interpretar ni evaluar expresiones `condition` del hook:
  - Si el hook no tiene campo `condition`, o es nulo/vacío, considéralo ejecutable.
  - Si el hook define una `condition` no vacía, omite el hook y deja la evaluación de la condición a la implementación de HookExecutor.
- Al construir invocaciones de comandos a partir de nombres de comandos de hooks, sustituye los puntos (`.`) por guiones (`-`). Por ejemplo, `speckit.git.commit` → `/speckit-git-commit`.
- Para cada hook ejecutable, muestra lo siguiente según el valor de su bandera `optional`:
  - **Hook opcional** (`optional: true`):
    ```
    ## Hooks de extensiones

    **Hook opcional**: {extension}
    Comando: `/{command}`
    Descripción: {description}

    Prompt: {prompt}
    Para ejecutar: `/{command}`
    ```
  - **Hook obligatorio** (`optional: false`):
    ```
    ## Hooks de extensiones

    **Hook automático**: {extension}
    Ejecutando: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.
- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.

## Principios operativos

### Eficiencia de contexto

- **Mínimos tokens de alto valor**: Concéntrate en hallazgos accionables, no en documentación exhaustiva.
- **Divulgación progresiva**: Carga los artefactos incrementalmente; no vuelques todo el contenido en el análisis.
- **Salida eficiente en tokens**: Limita la tabla de hallazgos a 50 filas; resume el excedente.
- **Resultados deterministas**: Volver a ejecutar sin cambios debería producir IDs y recuentos consistentes.

### Directrices de análisis

- **NUNCA modifiques archivos** (este análisis es de solo lectura).
- **NUNCA inventes secciones faltantes** (si no existen, repórtalo con precisión).
- **Prioriza las violaciones de la constitución** (siempre son CRITICAL).
- **Usa ejemplos en lugar de reglas exhaustivas** (cita instancias específicas, no patrones genéricos).
- **Informa correctamente cuando no haya problemas** (emite un informe satisfactorio con estadísticas de cobertura).

## Contexto

$ARGUMENTS
