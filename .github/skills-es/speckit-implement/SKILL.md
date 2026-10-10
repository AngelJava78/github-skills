---
name: "speckit-implement"
description: "Ejecutar el plan de implementación procesando y ejecutando todas las tareas definidas en tasks.md"
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/implement.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de la implementación)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_implement`.
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

2. **Comprobar el estado de las checklists** (si existe FEATURE_DIR/checklists/):
   - Trata los marcadores de checklist como un gate de solo lectura: analiza el estado de las casillas, informa el estado y pregunta antes de continuar cuando sea necesario; **NO** modifiques los archivos de checklist ni sus marcadores.
   - `checklists/requirements.md` es la checklist incorporada de calidad de especificación mantenida por `/speckit-specify` y `/speckit-clarify`; las checklists personalizadas generadas por `/speckit-checklist` son artefactos de revisión de calidad de requisitos cuyo responsable es el revisor.
   - Para checklists personalizadas, `[x]` significa que el revisor determinó que se cumple el criterio de calidad de requisitos; **NO** significa que el trabajo de implementación esté terminado.
   - Analiza todos los archivos de checklist del directorio checklists/.
   - Para cada checklist, cuenta:
     - Elementos totales: todas las líneas que coincidan con `- [ ]`, `- [X]` o `- [x]`.
     - Elementos marcados: líneas que coincidan con `- [X]` o `- [x]`.
     - Elementos sin marcar: líneas que coincidan con `- [ ]`.
   - Crea una tabla de estado:

     ```text
     | Checklist | Total | Marcados | Sin marcar | Estado |
     |-----------|-------|----------|------------|--------|
     | ux.md     | 12    | 12       | 0          | ✓ PASS |
     | test.md   | 8     | 5        | 3          | ✗ FAIL |
     | security.md | 6   | 6        | 0          | ✓ PASS |
     ```

   - Calcula el estado general:
     - **PASS**: Todas las checklists tienen 0 elementos sin marcar.
     - **FAIL**: Una o más checklists tienen elementos sin marcar.

   - **Si alguna checklist tiene elementos sin marcar**:
     - Muestra la tabla con el número de elementos sin marcar.
     - **DETENTE** y pregunta: "Algunas checklists tienen elementos sin marcar. ¿Quieres continuar con la implementación de todos modos? (sí/no)"
     - Espera la respuesta del usuario antes de continuar.
     - Si el usuario responde "no", "espera" o "detente", interrumpe la ejecución.
     - Si responde "sí", "proceder" o "continuar", pasa al paso 3.

   - **Si todas las checklists están marcadas**:
     - Muestra la tabla indicando que todas las checklists aprobaron.
     - Continúa automáticamente al paso 3.

3. Carga y analiza el contexto de implementación:
   - **OBLIGATORIO**: Lee tasks.md para obtener la lista completa de tareas y el plan de ejecución.
   - **OBLIGATORIO**: Lee plan.md para conocer stack tecnológico, arquitectura y estructura de archivos.
   - **SI EXISTE**: Lee data-model.md para entidades y relaciones.
   - **SI EXISTE**: Lee contracts/ para especificaciones de API y requisitos de pruebas.
   - **SI EXISTE**: Lee research.md para decisiones técnicas y restricciones.
   - **SI EXISTE**: Lee .specify/memory/constitution.md para restricciones de gobierno.
   - **SI EXISTE**: Lee quickstart.md para escenarios de integración.

4. **Verificación de configuración del proyecto**:
   - **OBLIGATORIO**: Crea/verifica archivos ignore basándose en la configuración real del proyecto.

   **Lógica de detección y creación**:
   - Comprueba si el siguiente comando se ejecuta correctamente para determinar si el repositorio es un repo Git (si lo es, crea/verifica .gitignore):

     ```sh
     git rev-parse --git-dir 2>/dev/null
     ```

   - Comprueba si existe Dockerfile* o si Docker aparece en plan.md → crea/verifica .dockerignore.
   - Comprueba si existe .eslintrc* → crea/verifica .eslintignore.
   - Comprueba si existe eslint.config.* → asegúrate de que las entradas `ignores` de la configuración cubran los patrones necesarios.
   - Comprueba si existe .prettierrc* → crea/verifica .prettierignore.
   - Comprueba si existe .npmrc o package.json → crea/verifica .npmignore (si se va a publicar).
   - Comprueba si existen archivos terraform (*.tf) → crea/verifica .terraformignore.
   - Comprueba si se necesita .helmignore (hay charts Helm) → crea/verifica .helmignore.

   **Si el archivo ignore ya existe**: Verifica que contenga los patrones esenciales y agrega únicamente los patrones críticos faltantes.
   **Si falta el archivo ignore**: Créalo con el conjunto completo de patrones para la tecnología detectada.

   **Patrones comunes por tecnología** (según el stack de plan.md):
   - **Node.js/JavaScript/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
   - **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `dist/`, `*.egg-info/`
   - **Java**: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
   - **C#/.NET**: `bin/`, `obj/`, `*.user`, `*.suo`, `packages/`
   - **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
   - **Ruby**: `.bundle/`, `log/`, `tmp/`, `*.gem`, `vendor/bundle/`
   - **PHP**: `vendor/`, `*.log`, `*.cache`, `*.env`
   - **Rust**: `target/`, `debug/`, `release/`, `*.rs.bk`, `*.rlib`, `*.prof*`, `.idea/`, `*.log`, `.env*`
   - **Kotlin**: `build/`, `out/`, `.gradle/`, `.idea/`, `*.class`, `*.jar`, `*.iml`, `*.log`, `.env*`
   - **C++**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.so`, `*.a`, `*.exe`, `*.dll`, `.idea/`, `*.log`, `.env*`
   - **C**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.a`, `*.so`, `*.exe`, `*.dll`, `autom4te.cache/`, `config.status`, `config.log`, `.idea/`, `*.log`, `.env*`
   - **Swift**: `.build/`, `DerivedData/`, `*.swiftpm/`, `Packages/`
   - **R**: `.Rproj.user/`, `.Rhistory`, `.RData`, `.Ruserdata`, `*.Rproj`, `packrat/`, `renv/`
   - **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`, `.vscode/`, `.idea/`

   **Patrones específicos de herramientas**:
   - **Docker**: `node_modules/`, `.git/`, `Dockerfile*`, `.dockerignore`, `*.log*`, `.env*`, `coverage/`
   - **ESLint**: `node_modules/`, `dist/`, `build/`, `coverage/`, `*.min.js`
   - **Prettier**: `node_modules/`, `dist/`, `build/`, `coverage/`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
   - **Terraform**: `.terraform/`, `*.tfstate*`, `*.tfvars`, `.terraform.lock.hcl`
   - **Kubernetes/k8s**: `*.secret.yaml`, `secrets/`, `.kube/`, `kubeconfig*`, `*.key`, `*.crt`

5. Analiza la estructura de tasks.md y extrae:
   - **Fases de tareas**: Configuración, Pruebas, Núcleo, Integración, Pulido.
   - **Dependencias de tareas**: Reglas de ejecución secuencial frente a paralela.
   - **Detalles de tareas**: ID, descripción, rutas de archivo, marcadores paralelos [P].
   - **Flujo de ejecución**: Orden y requisitos de dependencia.

6. Ejecuta la implementación siguiendo el plan de tareas:
   - **Ejecución fase por fase**: Completa cada fase antes de pasar a la siguiente.
   - **Respeta dependencias**: Ejecuta las tareas secuenciales en orden; las tareas paralelas [P] pueden ejecutarse conjuntamente.
   - **Sigue un enfoque TDD**: Ejecuta las tareas de pruebas antes que sus correspondientes tareas de implementación.
   - **Coordinación basada en archivos**: Las tareas que afecten a los mismos archivos deben ejecutarse secuencialmente.
   - **Puntos de control de validación**: Verifica la finalización de cada fase antes de continuar.

7. Reglas de ejecución de la implementación:
   - **Configuración primero**: Inicializa estructura del proyecto, dependencias y configuración.
   - **Pruebas antes del código**: Si necesitas escribir pruebas para contratos, entidades y escenarios de integración.
   - **Desarrollo del núcleo**: Implementa modelos, servicios, comandos CLI y endpoints.
   - **Trabajo de integración**: Conexiones a base de datos, middleware, logging y servicios externos.
   - **Pulido y validación**: Pruebas unitarias, optimización de rendimiento y documentación.

8. Seguimiento del progreso y manejo de errores:
   - Informa el progreso después de cada tarea completada.
   - Detén la ejecución si falla cualquier tarea no paralela.
   - Para tareas paralelas [P], continúa con las tareas exitosas e informa las fallidas.
   - Proporciona mensajes de error claros con contexto para depuración.
   - Sugiere siguientes pasos si la implementación no puede continuar.
   - **IMPORTANTE**: Para las tareas completadas, asegúrate de marcarlas como [X] en el archivo de tareas.

9. Validación de finalización:
   - Verifica que todas las tareas obligatorias estén completadas.
   - Comprueba que las funcionalidades implementadas coincidan con la especificación original.
   - Valida que las pruebas pasen y que la cobertura cumpla los requisitos.
   - Confirma que la implementación siga el plan técnico.

Nota: Este comando presupone que existe un desglose completo de tareas en tasks.md. Si las tareas están incompletas o faltan, sugiere ejecutar primero `/speckit-tasks` para regenerar la lista de tareas.

## Hooks obligatorios posteriores a la ejecución

**DEBES completar esta sección antes de informar al usuario que el proceso terminó.**

Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si no existe o no hay hooks registrados bajo `hooks.after_implement`, pasa al Informe de finalización.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_implement`.
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

Informa el estado final con un resumen del trabajo completado.

## Finalizado cuando

- [ ] Todas las tareas de tasks.md están completadas y marcadas como `[X]`.
- [ ] La implementación está validada contra la especificación, el plan y la cobertura de pruebas.
- [ ] Los hooks de extensiones se despacharon u omitieron conforme a las reglas de la sección Hooks obligatorios posteriores a la ejecución.
- [ ] Se informó al usuario la finalización con un resumen del trabajo completado.
