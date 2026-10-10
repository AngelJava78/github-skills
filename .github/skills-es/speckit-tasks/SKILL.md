---
name: "speckit-tasks"
description: "Generar un tasks.md accionable y ordenado por dependencias para la funcionalidad basándose en los artefactos de diseño disponibles."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/tasks.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de generar las tareas)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_tasks`.
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

1. **Configuración**: Ejecuta `.specify/scripts/powershell/setup-tasks.ps1 -Json` desde la raíz del repositorio y analiza FEATURE_DIR, TASKS_TEMPLATE_CONTENT, TASKS_TEMPLATE y la lista AVAILABLE_DOCS. `FEATURE_DIR` y `TASKS_TEMPLATE` deben ser rutas absolutas cuando se proporcionen. `AVAILABLE_DOCS` es una lista de nombres de documentos/rutas relativas disponibles bajo `FEATURE_DIR` (por ejemplo, `research.md` o `contracts/`). Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

2. **Cargar documentos de diseño**: Lee desde FEATURE_DIR:
   - **Obligatorios**: plan.md (stack tecnológico, librerías, estructura), spec.md (historias de usuario con prioridades).
   - **Opcionales**: data-model.md (entidades), contracts/ (contratos de interfaz), research.md (decisiones), quickstart.md (escenarios de prueba).
   - **SI EXISTE**: Carga `.specify/memory/constitution.md` para obtener los principios del proyecto y las restricciones de gobierno.
   - Nota: No todos los proyectos tienen todos los documentos. Genera las tareas basándote en lo que esté disponible.

3. **Ejecutar el flujo de generación de tareas**:
   - Carga plan.md y extrae stack tecnológico, librerías y estructura del proyecto.
   - Carga spec.md y extrae historias de usuario con sus prioridades (P1, P2, P3, etc.).
   - Si existe data-model.md: extrae las entidades y asócialas con historias de usuario.
   - Si existe contracts/: asocia los contratos de interfaz con historias de usuario.
   - Si existe research.md: extrae decisiones para las tareas de configuración.
   - Genera tareas organizadas por historia de usuario (consulta las Reglas de generación de tareas más abajo).
   - Genera un grafo de dependencias que muestre el orden de finalización de las historias de usuario.
   - Crea ejemplos de ejecución paralela por historia de usuario.
   - Valida la completitud de las tareas (cada historia de usuario tiene todas las tareas necesarias y puede probarse de forma independiente).

4. **Generar tasks.md**: Usa TASKS_TEMPLATE_CONTENT (de la salida JSON anterior) como estructura. Para mantener compatibilidad con scripts de configuración antiguos que omitan TASKS_TEMPLATE_CONTENT, lee TASKS_TEMPLATE en su lugar. Complétalo con:
   - Nombre correcto de la funcionalidad tomado de plan.md.
   - Fase 1: Tareas de configuración (inicialización del proyecto).
   - Fase 2: Tareas fundamentales (prerrequisitos bloqueantes para todas las historias de usuario).
   - Fase 3+: Una fase por historia de usuario (en el orden de prioridad de spec.md).
   - Cada fase incluye: objetivo de la historia, criterios de prueba independiente, pruebas (si se solicitaron), tareas de implementación.
   - Fase final: Pulido y aspectos transversales.
   - Todas las tareas deben seguir estrictamente el formato de checklist (consulta las Reglas de generación de tareas más abajo).
   - Rutas de archivo claras para cada tarea.
   - Sección de dependencias que muestre el orden de finalización de las historias.
   - Ejemplos de ejecución paralela por historia.
   - Sección de estrategia de implementación (MVP primero, entrega incremental).

## Hooks obligatorios posteriores a la ejecución

**DEBES completar esta sección antes de informar al usuario que el proceso terminó.**

Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si no existe o no hay hooks registrados bajo `hooks.after_tasks`, pasa al Informe de finalización.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_tasks`.
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

Muestra la ruta del tasks.md generado y un resumen:
- Número total de tareas.
- Número de tareas por historia de usuario.
- Oportunidades de paralelismo identificadas.
- Criterios de prueba independiente para cada historia.
- Alcance sugerido del MVP (normalmente solo la Historia de Usuario 1).
- Validación del formato: confirma que **TODAS** las tareas siguen el formato de checklist (checkbox, ID, etiquetas, rutas de archivo).

Contexto para generación de tareas: $ARGUMENTS

El tasks.md debe poder ejecutarse inmediatamente: cada tarea debe ser lo bastante específica para que un LLM pueda completarla sin contexto adicional.

## Reglas de generación de tareas

**CRÍTICO**: Las tareas **DEBEN** organizarse por historia de usuario para permitir implementación y pruebas independientes.

**Las pruebas son OPCIONALES**: Genera tareas de prueba únicamente si se solicitan explícitamente en la especificación de la funcionalidad o si el usuario solicita un enfoque TDD.

### Formato de checklist (OBLIGATORIO)

Cada tarea **DEBE** seguir estrictamente este formato:

```text
- [ ] [TaskID] [P?] [Story?] Descripción con ruta de archivo
```

**Componentes del formato**:

1. **Checkbox**: Comienza **SIEMPRE** con `- [ ]` (checkbox Markdown).
2. **ID de tarea**: Número secuencial (T001, T002, T003...) en orden de ejecución.
3. **Marcador [P]**: Inclúyelo **SOLO** si la tarea puede ejecutarse en paralelo (archivos distintos, sin dependencias respecto de tareas incompletas).
4. **Etiqueta [Story]**: OBLIGATORIA únicamente para tareas de fases de historias de usuario.
   - Formato: [US1], [US2], [US3], etc. (corresponde a las historias de usuario de spec.md).
   - Fase Setup: SIN etiqueta de historia.
   - Fase Foundational: SIN etiqueta de historia.
   - Fases de Historias de Usuario: **DEBEN** tener etiqueta de historia.
   - Fase Polish: SIN etiqueta de historia.
5. **Descripción**: Acción clara con ruta exacta del archivo.

**Ejemplos**:

- ✅ CORRECTO: `- [ ] T001 Create project structure per implementation plan`
- ✅ CORRECTO: `- [ ] T005 [P] Implement authentication middleware in src/middleware/auth.py`
- ✅ CORRECTO: `- [ ] T012 [P] [US1] Create User model in src/models/user.py`
- ✅ CORRECTO: `- [ ] T014 [US1] Implement UserService in src/services/user_service.py`
- ❌ INCORRECTO: `- [ ] Create User model` (falta ID y etiqueta Story)
- ❌ INCORRECTO: `T001 [US1] Create model` (falta checkbox)
- ❌ INCORRECTO: `- [ ] [US1] Create User model` (falta ID de tarea)
- ❌ INCORRECTO: `- [ ] T001 [US1] Create model` (falta ruta de archivo)

### Organización de tareas

1. **A partir de Historias de Usuario (spec.md)** - ORGANIZACIÓN PRINCIPAL:
   - Cada historia de usuario (P1, P2, P3...) obtiene su propia fase.
   - Asocia todos los componentes relacionados con su historia:
     - Modelos necesarios para esa historia.
     - Servicios necesarios para esa historia.
     - Interfaces/UI necesarias para esa historia.
     - Si se solicitaron pruebas: pruebas específicas de esa historia.
   - Marca las dependencias entre historias (la mayoría deberían ser independientes).

2. **A partir de Contratos**:
   - Asocia cada contrato de interfaz → con la historia de usuario a la que sirve.
   - Si se solicitaron pruebas: cada contrato de interfaz → tarea de prueba de contrato [P] antes de la implementación en la fase de esa historia.

3. **A partir del Modelo de Datos**:
   - Asocia cada entidad con la(s) historia(s) de usuario que la necesitan.
   - Si una entidad sirve a varias historias: colócala en la historia más temprana o en la fase Setup.
   - Relaciones → tareas de capa de servicios en la fase de historia correspondiente.
   - Para cada campo con restricciones en data-model.md (longitud máxima, nullable/required, valores enum, reglas de validación), cita la restricción literalmente en la descripción de la tarea para que no quede a criterio durante la implementación.

4. **A partir de Configuración/Infraestructura**:
   - Infraestructura compartida → fase Setup (Fase 1).
   - Tareas fundamentales/bloqueantes → fase Foundational (Fase 2).
   - Configuración específica de una historia → dentro de la fase de esa historia.

### Estructura de fases

- **Fase 1**: Setup (inicialización del proyecto).
- **Fase 2**: Foundational (prerrequisitos bloqueantes: **DEBEN** completarse antes de las historias de usuario).
- **Fase 3+**: Historias de Usuario en orden de prioridad (P1, P2, P3...).
  - Dentro de cada historia: Pruebas (si se solicitaron) → Modelos → Servicios → Endpoints → Integración.
  - Cada fase debe ser un incremento completo y comprobable de forma independiente.
- **Fase final**: Polish & Cross-Cutting Concerns.

## Finalizado cuando

- [ ] tasks.md se generó con todas las fases, IDs de tareas y rutas de archivo.
- [ ] Los hooks de extensiones se despacharon u omitieron conforme a las reglas de la sección Hooks obligatorios posteriores a la ejecución.
- [ ] Se informó al usuario la finalización con el número de tareas, desglose por historia y alcance del MVP.
