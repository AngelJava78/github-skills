---
name: "speckit-taskstoissues"
description: "Convertir las tareas existentes en issues de GitHub accionables y ordenados por dependencias para la funcionalidad, basándose en los artefactos de diseño disponibles."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/taskstoissues.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de convertir tareas en issues)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_taskstoissues`.
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

1. Ejecuta `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks` desde la raíz del repositorio y analiza FEATURE_DIR y la lista AVAILABLE_DOCS. Todas las rutas deben ser absolutas. Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").
1. **SI EXISTE**: Carga `.specify/memory/constitution.md` para obtener los principios del proyecto y las restricciones de gobierno.
1. A partir del script ejecutado, extrae la ruta a **tasks**.
1. Obtén el remote de Git ejecutando:

```bash
git config --get remote.origin.url
```

> [!CAUTION]
> CONTINÚA CON LOS SIGUIENTES PASOS ÚNICAMENTE SI EL REMOTE ES UNA URL DE GITHUB

1. **Recuperar issues existentes para evitar duplicados**: Antes de crear nada, construye el conjunto de IDs de tareas que vas a procesar desde `tasks.md` (cada uno es una `T` seguida de **al menos** tres dígitos, por ejemplo `T001`; `/speckit-converge` asigna nuevos IDs como `T{M+1:03d}`, lo cual establece un mínimo y no un máximo, por lo que cuando un archivo supera las 999 tareas los IDs tienen cuatro o más dígitos). Luego usa la herramienta `list_issues` del servidor MCP de GitHub para buscar issues que ya cubran esos IDs. No envíes un valor `state`, ya que omitirlo hace que la herramienta devuelva tanto issues abiertos como cerrados. Solicita `perPage: 100` para reducir el número de llamadas y, dado que la herramienta usa paginación basada en cursor, solicita páginas con el parámetro `after` (usando el `endCursor` de la respuesta anterior). Para cada título de issue, compáralo con el patrón de ID de tarea `\bT\d{3,}\b` (el `{3,}` acepta IDs de cuatro o más dígitos; con `\d{3}` un título que contenga `T1000` no coincidiría en absoluto, porque el `\b` final no puede situarse entre dos dígitos, por lo que esa tarea quedaría silenciosamente sin deduplicar ni crear; los límites de palabra también impiden que un token como `ST001` coincida y obligan a consumir toda la secuencia de dígitos, de modo que `T100` nunca pueda coincidir dentro de `T1000`; esto también reconoce títulos escritos como `T001 ...`, `T001: ...` o `[T001] ...`). Cuando coincida con uno de tus IDs de tarea, marca ese ID como uno que ya tiene issue. Detén la paginación tan pronto como se hayan encontrado todos los IDs de tarea o cuando no haya más páginas, para no seguir recuperando todo el historial de issues del repositorio una vez que todos los IDs estén cubiertos. Esto limita el número de llamadas en repositorios con historiales grandes y sigue evitando duplicados cuando se vuelve a ejecutar el comando después de regenerar `tasks.md` o de volver a invocar el skill.
1. Para cada tarea de la lista, usa el servidor MCP de GitHub para crear un nuevo issue en el repositorio representado por el remote de Git. Las líneas de tareas en `tasks.md` comienzan con una casilla Markdown, por lo que primero elimina el prefijo `- [ ]` (y cualquier marcador `[P]` / `[US#]`) para recuperar el ID de tarea y su descripción. Crea el issue con un único título canónico con la forma `T001: <description>`, escribiendo el ID una sola vez seguido de la descripción de la tarea (por ejemplo, la línea `- [ ] T001 Create project structure` se convierte en el título `T001: Create project structure`).
   - **Omite** cualquier tarea cuyo ID ya exista en el conjunto de issues del paso anterior e infórmalo (por ejemplo, `T001 already has an issue, skipping`).
   - Crea issues únicamente para las tareas que todavía no tengan un issue coincidente.

> [!CAUTION]
> BAJO NINGUNA CIRCUNSTANCIA CREES ISSUES EN REPOSITORIOS QUE NO COINCIDAN CON LA URL DEL REMOTE

## Comprobaciones posteriores a la ejecución

**Comprobar hooks de extensiones (después de convertir tareas en issues)**:
Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_taskstoissues`.
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
