---
name: "speckit-constitution"
description: "Crear o actualizar la constitución del proyecto a partir de principios proporcionados por el usuario o recopilados de forma interactiva."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/constitution.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Protección de alcance

El trabajo de este comando se limita a actualizar la constitución del proyecto. Las plantillas y comandos dependientes leen la constitución en tiempo de ejecución y no se modifican aquí.

- Clasifica cada parte de la entrada del usuario como contenido de la constitución o como una intención separada no relacionada con gobierno.
- Si la entrada incluye solicitudes de implementación de funcionalidades, generación de código, refactorización, construcción o despliegue, **NO DEBES** ejecutarlas. Extráelas como intenciones diferidas.
- **NO DEBES** crear, modificar ni eliminar archivos de código fuente de la aplicación, rutas de funcionalidades, componentes, pruebas, archivos de despliegue ni otros artefactos ajenos al flujo de constitución.
- Si no está claro si una instrucción corresponde a contenido de la constitución, solicita aclaración antes de realizar cambios.
- Después de completar la actualización de la constitución, incluye una sección `Next Actions` por cada intención diferida. Enumera la intención original y sugiere el comando de Spec Kit adecuado para continuar, como `/speckit-specify`, sin invocarlo.
- Si no existen intenciones no relacionadas con gobierno, omite la sección `Next Actions`.

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de actualizar la constitución)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_constitution`.
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

Estás actualizando la constitución del proyecto en `.specify/memory/constitution.md`. La estructura activa de la constitución se resuelve en el momento de ejecutar el comando a partir de `constitution-template`, mediante la pila de resolución de presets/plantillas de Spec Kit.

Sigue este flujo de ejecución:

1. Ejecuta `.specify/scripts/powershell/resolve-template.ps1 constitution-template -Json` desde la raíz del repositorio y analiza `TEMPLATE_CONTENT` como plantilla activa.
   - El resolver compartido aplica overrides del proyecto, capas de presets compuestas y capas de extensiones antes de recurrir a la plantilla base. **DEBE** completarse correctamente antes de continuar.
   - Si falla, detente e informa el error de resolución; no continúes usando únicamente una de las capas que contribuyen a la plantilla.
   - Si `.specify/memory/constitution.md` existe, cárgalo como fuente de los valores y enmiendas actuales específicos del proyecto. Conserva la información que siga siendo aplicable al usar la nueva estructura resuelta.
   - Si no existe, usa la plantilla resuelta como documento inicial.
   - No escribas cambios en ninguna capa de plantillas versionadas.
   - Identifica todos los tokens de marcador con la forma `[ALL_CAPS_IDENTIFIER]`.
   **IMPORTANTE**: El usuario puede requerir menos o más principios que los utilizados en la plantilla. Si especifica una cantidad, respétala siguiendo la estructura general de la plantilla. Actualiza el documento en consecuencia.

2. Recopila/deriva valores para los marcadores:
   - Si la entrada del usuario (conversación) proporciona un valor, úsalo.
   - En caso contrario, infiérelo del contexto existente del repositorio (README, documentación, versiones previas de la constitución si están incorporadas).
   - Para fechas de gobierno: `RATIFICATION_DATE` es la fecha de adopción original (si se desconoce, pregunta o marca TODO); `LAST_AMENDED_DATE` es la fecha de hoy si se realizan cambios; de lo contrario, conserva la anterior.
   - `CONSTITUTION_VERSION` debe incrementarse conforme a las reglas de versionado semántico:
     - MAJOR: Cambios incompatibles hacia atrás en gobierno/principios, eliminaciones o redefiniciones.
     - MINOR: Nuevo principio/sección o ampliación material de directrices.
     - PATCH: Aclaraciones, redacción, correcciones tipográficas o refinamientos no semánticos.
   - Si el tipo de incremento de versión es ambiguo, propón el razonamiento antes de finalizar.

3. Redacta el contenido actualizado de la constitución usando la plantilla resuelta como estructura obligatoria:
   - Sustituye cada marcador por texto concreto (no deben quedar tokens entre corchetes, salvo slots de plantilla que el proyecto haya decidido dejar deliberadamente sin definir; justifica explícitamente cualquiera que permanezca).
   - Conserva la jerarquía de encabezados; los comentarios pueden eliminarse después de sustituirse salvo que sigan aportando orientación aclaratoria.
   - Asegúrate de que cada sección de Principio contenga: nombre conciso, un párrafo (o lista) con reglas no negociables y una justificación explícita cuando no sea obvia.
   - Asegúrate de que la sección de Gobierno incluya el procedimiento de enmienda, la política de versionado y las expectativas de revisión de cumplimiento.

4. Genera un Informe de impacto de sincronización como comentario HTML al inicio del archivo de constitución después de la actualización.
   Este informe es material temporal para la revisión humana de la enmienda, no contenido de gobierno; se espera que se elimine antes de hacer commit del archivo de constitución modificado.
   - Cambio de versión: anterior → nueva.
   - Lista de principios modificados (título anterior → título nuevo, si se renombró).
   - Secciones agregadas.
   - Secciones eliminadas.
   - TODOs de seguimiento si algún marcador se dejó intencionalmente pendiente.

5. Validación antes de la salida final:
   - No deben quedar tokens entre corchetes sin explicación.
   - La línea de versión debe coincidir con el informe.
   - Las fechas deben usar formato ISO YYYY-MM-DD.
   - Los principios deben ser declarativos, comprobables y evitar lenguaje vago ("should" → sustituir por MUST/SHOULD con justificación cuando corresponda).

6. Escribe la constitución completada de vuelta en `.specify/memory/constitution.md` (sobrescribe el archivo).

7. Genera un resumen final para el usuario con:
   - Nueva versión y justificación del incremento.
   - Cualquier marcador TODO o elemento diferido que requiera seguimiento manual.
   - Mensaje de commit sugerido (por ejemplo, `docs: amend constitution to vX.Y.Z (principle additions + governance update)`).
   - Una sección `Next Actions` para cualquier intención diferida no relacionada con gobierno.

Requisitos de formato y estilo:

- Usa los encabezados Markdown exactamente como aparecen en la plantilla (no subas ni bajes niveles).
- Ajusta las líneas largas de justificación para mantener la legibilidad (<100 caracteres idealmente), pero no fuerces cortes incómodos.
- Mantén una sola línea en blanco entre secciones.
- Evita espacios al final de línea.

Si el usuario proporciona actualizaciones parciales (por ejemplo, solo la revisión de un principio), realiza igualmente los pasos de validación y decisión de versión.

Si falta información crítica (por ejemplo, se desconoce realmente la fecha de ratificación), inserta `TODO(<FIELD_NAME>): explicación` e inclúyelo en el Informe de impacto de sincronización dentro de los elementos diferidos.

Escribe únicamente `.specify/memory/constitution.md`; no crees ni modifiques archivos fuente de plantillas.

## Comprobaciones posteriores a la ejecución

**Comprobar hooks de extensiones (después de actualizar la constitución)**:
Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_constitution`.
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
