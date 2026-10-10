---
name: "speckit-specify"
description: "Crear o actualizar la especificación de una funcionalidad a partir de una descripción en lenguaje natural."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/specify.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de la especificación)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_specify`.
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

El texto que el usuario escribió después de `/speckit-specify` en el mensaje que activó el comando **es** la descripción de la funcionalidad. Asume que siempre está disponible en esta conversación aunque `$ARGUMENTS` aparezca literalmente más abajo. No pidas al usuario que la repita salvo que haya proporcionado un comando vacío.

Dada esa descripción de la funcionalidad, haz lo siguiente:

1. **Generar un nombre corto y conciso** (2-4 palabras) para la funcionalidad:
   - Analiza la descripción y extrae las palabras clave más relevantes.
   - Crea un nombre corto de 2-4 palabras que capture la esencia de la funcionalidad.
   - Usa el formato acción-sustantivo cuando sea posible (por ejemplo, "add-user-auth", "fix-payment-bug").
   - Conserva términos técnicos y acrónimos (OAuth2, API, JWT, etc.).
   - Mantenlo conciso, pero suficientemente descriptivo para entender la funcionalidad de un vistazo.
   - Ejemplos:
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **Creación de rama** (opcional, mediante hook):

   Si un hook `before_specify` se ejecutó correctamente en las Comprobaciones previas a la ejecución, habrá creado/cambiado a una rama Git y habrá generado JSON con `BRANCH_NAME` y `FEATURE_NUM`. Conserva estos valores como referencia, pero el nombre de la rama **no** determina el nombre del directorio de la especificación.

   Si el usuario proporcionó explícitamente `GIT_BRANCH_NAME`, pásalo al hook para que el script de rama use exactamente ese valor como nombre de rama (sin generar prefijos/sufijos).

3. **Crear el directorio de la funcionalidad de especificación**:

   Las especificaciones viven bajo el directorio predeterminado `specs/`, salvo que el usuario proporcione explícitamente `SPECIFY_FEATURE_DIRECTORY`.

   **Orden de resolución de `SPECIFY_FEATURE_DIRECTORY`**:
   1. Si el usuario proporcionó explícitamente `SPECIFY_FEATURE_DIRECTORY` (por ejemplo, mediante variable de entorno, argumento o configuración), úsalo tal cual.
   2. En caso contrario, genéralo automáticamente bajo `specs/`:
      - Comprueba `.specify/init-options.json` para `feature_numbering` (preferido) o `branch_numbering` (obsoleto, solo para migración; se eliminará en una versión futura).
      - Si es `"timestamp"`: el prefijo es `YYYYMMDD-HHMMSS` (timestamp actual).
      - Si es `"sequential"` o está ausente: el prefijo es `NNN` (el siguiente número disponible de 3 dígitos después de recorrer los directorios existentes en `specs/`).
      - Construye el nombre del directorio: `<prefix>-<short-name>` (por ejemplo, `003-user-auth` o `20260319-143022-user-auth`).
      - Establece `SPECIFY_FEATURE_DIRECTORY` a `specs/<directory-name>`.
      - Si se utilizó `branch_numbering` (y `feature_numbering` estaba ausente), muestra una advertencia de una línea: "⚠️ `branch_numbering` in init-options.json is deprecated. Rename to `feature_numbering`."

   **Crear el directorio y el archivo de especificación**:
   - `mkdir -p SPECIFY_FEATURE_DIRECTORY`
   - Resuelve el `spec-template` activo mediante la pila de resolución de presets/plantillas de Spec Kit (equivalente a `specify preset resolve spec-template`).
   - Copia el archivo `spec-template` resuelto a `SPECIFY_FEATURE_DIRECTORY/spec.md` como punto de partida.
   - Establece `SPEC_FILE` a `SPECIFY_FEATURE_DIRECTORY/spec.md`.
   - Persiste la ruta resuelta en `.specify/feature.json`:
     ```json
     {
       "feature_directory": "<resolved feature dir>"
     }
     ```
     Escribe el valor real de la ruta del directorio resuelto (por ejemplo, `specs/003-user-auth`), no la cadena literal `SPECIFY_FEATURE_DIRECTORY`.
     Esto permite que los comandos posteriores (`/speckit-plan`, `/speckit-tasks`, etc.) localicen el directorio de la funcionalidad sin depender de convenciones de nombres de ramas Git.

   **IMPORTANTE**:
   - Debes crear únicamente una funcionalidad por cada invocación de `/speckit-specify`.
   - El nombre del directorio de especificación y el nombre de la rama Git son independientes; pueden ser iguales, pero eso depende de la elección del usuario.
   - Este comando siempre crea el directorio y el archivo de especificación; nunca lo hace el hook.

4. Carga el archivo `spec-template` activo resuelto para comprender las secciones obligatorias.

5. **SI EXISTE**: Carga `.specify/memory/constitution.md` para obtener los principios del proyecto y las restricciones de gobierno.

6. Sigue este flujo de ejecución:
    1. Analiza la descripción del usuario a partir de los argumentos.
       Si está vacía: ERROR "No feature description provided".
    2. Extrae conceptos clave de la descripción.
       Identifica: actores, acciones, datos, restricciones.
    3. Para aspectos poco claros:
       - Haz inferencias informadas basándote en el contexto y estándares de la industria.
       - Marca con `[NEEDS CLARIFICATION: specific question]` únicamente si:
         - La elección afecta significativamente al alcance de la funcionalidad o a la experiencia del usuario.
         - Existen varias interpretaciones razonables con implicaciones diferentes.
         - No existe un valor predeterminado razonable.
       - **LÍMITE: Máximo 3 marcadores [NEEDS CLARIFICATION] en total**.
       - Prioriza las aclaraciones por impacto: alcance > seguridad/privacidad > experiencia de usuario > detalles técnicos.
    4. Completa la sección User Scenarios & Testing.
       Si no hay un flujo de usuario claro: ERROR "Cannot determine user scenarios".
    5. Genera Requisitos Funcionales.
       Cada requisito debe poder probarse.
       Usa valores predeterminados razonables para detalles no especificados (documenta los supuestos en la sección Assumptions).
    6. Define Criterios de Éxito.
       Crea resultados medibles e independientes de la tecnología.
       Incluye métricas cuantitativas (tiempo, rendimiento, volumen) y medidas cualitativas (satisfacción del usuario, finalización de tareas).
       Cada criterio debe poder verificarse sin detalles de implementación.
    7. Identifica Entidades Clave (si hay datos involucrados).
    8. Devuelve: SUCCESS (especificación lista para planificación).

7. Escribe la especificación en SPEC_FILE usando la estructura de la plantilla, sustituyendo los marcadores con detalles concretos derivados de la descripción de la funcionalidad (argumentos), conservando el orden de secciones y encabezados.

8. **Validación de calidad de la especificación**: Después de escribir la especificación inicial, valídala contra criterios de calidad:

   a. **Crear checklist de calidad de la especificación**: Genera un archivo de checklist en `SPECIFY_FEATURE_DIRECTORY/checklists/requirements.md` usando la estructura de la plantilla de checklist con estos elementos de validación:

      ```markdown
      # Checklist de calidad de la especificación: [FEATURE NAME]

      **Propósito**: Validar la completitud y calidad de la especificación antes de continuar con la planificación
      **Creado**: [DATE]
      **Funcionalidad**: [Link to spec.md]

      ## Calidad del contenido

      - [ ] Sin detalles de implementación (lenguajes, frameworks, APIs)
      - [ ] Enfocada en el valor para el usuario y las necesidades del negocio
      - [ ] Redactada para stakeholders no técnicos
      - [ ] Todas las secciones obligatorias están completas

      ## Completitud de requisitos

      - [ ] No quedan marcadores [NEEDS CLARIFICATION]
      - [ ] Los requisitos son comprobables e inequívocos
      - [ ] Los criterios de éxito son medibles
      - [ ] Los criterios de éxito son independientes de la tecnología (sin detalles de implementación)
      - [ ] Todos los escenarios de aceptación están definidos
      - [ ] Los casos límite están identificados
      - [ ] El alcance está claramente delimitado
      - [ ] Las dependencias y supuestos están identificados

      ## Preparación de la funcionalidad

      - [ ] Todos los requisitos funcionales tienen criterios de aceptación claros
      - [ ] Los escenarios de usuario cubren los flujos principales
      - [ ] La funcionalidad cumple los resultados medibles definidos en Criterios de Éxito
      - [ ] No se filtran detalles de implementación en la especificación

      ## Notas

      - Los elementos marcados como incompletos requieren actualizaciones de la especificación antes de `/speckit-clarify` o `/speckit-plan`
      ```

   b. **Ejecutar comprobación de validación**: Revisa la especificación contra cada elemento de la checklist:
      - Para cada elemento, determina si aprueba o falla.
      - Documenta los problemas específicos encontrados (cita las secciones relevantes de la especificación).

   c. **Gestionar los resultados de validación**:

      - **Si todos los elementos aprueban**: Marca la checklist como completa y continúa con la sección Hooks obligatorios posteriores a la ejecución.

      - **Si fallan elementos (excluyendo [NEEDS CLARIFICATION])**:
        1. Enumera los elementos fallidos y los problemas específicos.
        2. Actualiza la especificación para resolver cada problema.
        3. Vuelve a ejecutar la validación hasta que todos aprueben (máximo 3 iteraciones).
        4. Si siguen existiendo fallos después de 3 iteraciones, documenta los problemas restantes en las notas de la checklist y advierte al usuario.

      - **Si permanecen marcadores [NEEDS CLARIFICATION]**:
        1. Extrae todos los marcadores [NEEDS CLARIFICATION: ...] de la especificación.
        2. **COMPROBACIÓN DE LÍMITE**: Si existen más de 3 marcadores, conserva únicamente los 3 más críticos (por impacto de alcance/seguridad/UX) y haz inferencias informadas para el resto.
        3. Para cada aclaración necesaria (máx. 3), presenta opciones al usuario con este formato:

           ```markdown
           ## Pregunta [N]: [Tema]

           **Contexto**: [Cita de la sección relevante de la especificación]

           **Qué necesitamos saber**: [Pregunta específica del marcador NEEDS CLARIFICATION]

           **Respuestas sugeridas**:

           | Opción | Respuesta | Implicaciones |
           |--------|-----------|---------------|
           | A      | [Primera respuesta sugerida] | [Qué significa esto para la funcionalidad] |
           | B      | [Segunda respuesta sugerida] | [Qué significa esto para la funcionalidad] |
           | C      | [Tercera respuesta sugerida] | [Qué significa esto para la funcionalidad] |
           | Personalizada | Proporciona tu propia respuesta | [Explica cómo proporcionar una entrada personalizada] |

           **Tu elección**: _[Esperar respuesta del usuario]_
           ```

        4. **CRÍTICO - Formato de tablas**: Asegúrate de que las tablas Markdown estén correctamente formateadas:
           - Usa espaciado consistente con los pipes alineados.
           - Cada celda debe tener espacios alrededor del contenido: `| Contenido |`, no `|Contenido|`.
           - El separador del encabezado debe tener al menos 3 guiones: `|--------|`.
           - Comprueba que la tabla se renderice correctamente en la vista previa Markdown.
        5. Numera las preguntas secuencialmente (Q1, Q2, Q3; máximo 3 en total).
        6. Presenta todas las preguntas juntas antes de esperar respuestas.
        7. Espera a que el usuario responda con sus elecciones para todas las preguntas (por ejemplo, "Q1: A, Q2: Custom - [details], Q3: B").
        8. Actualiza la especificación sustituyendo cada marcador [NEEDS CLARIFICATION] con la respuesta seleccionada o proporcionada por el usuario.
        9. Vuelve a ejecutar la validación después de resolver todas las aclaraciones.

   d. **Actualizar checklist**: Después de cada iteración de validación, actualiza el archivo de checklist con el estado actual de aprobación/fallo.

## Hooks obligatorios posteriores a la ejecución

**DEBES completar esta sección antes de informar al usuario que el proceso terminó.**

Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si no existe o no hay hooks registrados bajo `hooks.after_specify`, pasa al Informe de finalización.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_specify`.
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

Informa al usuario que el proceso terminó e incluye:
- `SPECIFY_FEATURE_DIRECTORY` — ruta del directorio de la funcionalidad.
- `SPEC_FILE` — ruta del archivo de especificación.
- Resumen de resultados de la checklist.
- Preparación para la siguiente fase (`/speckit-clarify` o `/speckit-plan`).

**NOTA:** La creación de la rama la gestiona el hook `before_specify` (extensión Git). La creación del directorio y archivo de especificación siempre la gestiona este comando principal.

## Directrices rápidas

- Concéntrate en **QUÉ** necesitan los usuarios y **POR QUÉ**.
- Evita CÓMO implementarlo (sin stack tecnológico, APIs ni estructura de código).
- Redacta para stakeholders de negocio, no para desarrolladores.
- **NO** crees checklists incrustadas en la especificación. Eso corresponde a un comando separado.

### Requisitos de las secciones

- **Secciones obligatorias**: Deben completarse para toda funcionalidad.
- **Secciones opcionales**: Inclúyelas solo cuando sean relevantes para la funcionalidad.
- Cuando una sección no aplique, elimínala por completo (no la dejes como "N/A").

### Para generación con IA

Al crear esta especificación a partir de un prompt del usuario:

1. **Haz inferencias informadas**: Usa el contexto, estándares de la industria y patrones comunes para completar brechas.
2. **Documenta supuestos**: Registra valores predeterminados razonables en la sección Assumptions.
3. **Limita las aclaraciones**: Máximo 3 marcadores [NEEDS CLARIFICATION]; úsalos solo para decisiones críticas que:
   - Afecten significativamente al alcance de la funcionalidad o a la experiencia del usuario.
   - Tengan varias interpretaciones razonables con implicaciones distintas.
   - No dispongan de ningún valor predeterminado razonable.
4. **Prioriza aclaraciones**: alcance > seguridad/privacidad > experiencia del usuario > detalles técnicos.
5. **Piensa como tester**: Todo requisito vago debería fallar el elemento de checklist "comprobable e inequívoco".
6. **Áreas comunes que necesitan aclaración** (solo si no existe un valor predeterminado razonable):
   - Alcance y límites de la funcionalidad (incluir/excluir casos de uso específicos).
   - Tipos de usuario y permisos (si existen varias interpretaciones conflictivas posibles).
   - Requisitos de seguridad/cumplimiento (cuando tengan relevancia legal/financiera).

**Ejemplos de valores predeterminados razonables** (no preguntes por estos):

- Retención de datos: prácticas estándar de la industria para el dominio.
- Objetivos de rendimiento: expectativas estándar para aplicaciones web/móviles salvo que se especifique otra cosa.
- Manejo de errores: mensajes fáciles de entender para el usuario con mecanismos alternativos apropiados.
- Método de autenticación: sesiones estándar u OAuth2 para aplicaciones web.
- Patrones de integración: usa patrones apropiados para el proyecto (REST/GraphQL para servicios web, llamadas a funciones para librerías, argumentos CLI para herramientas, etc.).

### Directrices para criterios de éxito

Los criterios de éxito deben ser:

1. **Medibles**: Incluir métricas específicas (tiempo, porcentaje, cantidad, tasa).
2. **Independientes de la tecnología**: No mencionar frameworks, lenguajes, bases de datos ni herramientas.
3. **Centrados en el usuario**: Describir resultados desde la perspectiva del usuario/negocio, no de componentes internos del sistema.
4. **Verificables**: Poder probarse/validarse sin conocer detalles de implementación.

**Buenos ejemplos**:

- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"
- "95% of searches return results in under 1 second"
- "Task completion rate improves by 40%"

**Malos ejemplos** (centrados en implementación):

- "API response time is under 200ms" (demasiado técnico; usa "Users see results instantly").
- "Database can handle 1000 TPS" (detalle de implementación; usa una métrica visible para el usuario).
- "React components render efficiently" (específico de framework).
- "Redis cache hit rate above 80%" (específico de tecnología).

## Finalizado cuando

- [ ] La especificación se escribió en `SPEC_FILE` y se validó contra la checklist de calidad.
- [ ] Los hooks de extensiones se despacharon u omitieron conforme a las reglas de la sección Hooks obligatorios posteriores a la ejecución.
- [ ] Se informó al usuario la finalización con el directorio de la funcionalidad, la ruta del archivo de especificación y los resultados de la checklist.
