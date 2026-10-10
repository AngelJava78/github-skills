---
name: "speckit-plan"
description: "Ejecutar el flujo de planificación de la implementación usando la plantilla de plan para generar artefactos de diseño."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/plan.md"
---


## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de la planificación)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_plan`.
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

1. **Configuración**: Ejecuta `.specify/scripts/powershell/setup-plan.ps1 -Json` desde la raíz del repositorio y analiza el JSON para obtener FEATURE_SPEC, IMPL_PLAN, FEATURE_DIR y BRANCH. Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

2. **Cargar contexto**: Lee FEATURE_SPEC y `.specify/memory/constitution.md`. Carga la plantilla IMPL_PLAN (ya copiada).

3. **Ejecutar el flujo del plan**: Sigue la estructura de la plantilla IMPL_PLAN para:
   - Completar el Contexto técnico (marca los desconocidos como "NEEDS CLARIFICATION").
   - Completar la sección de Comprobación de la constitución a partir de la constitución.
   - Evaluar los gates (ERROR si existen violaciones no justificadas).
   - Fase 0: Generar research.md (resolver todos los NEEDS CLARIFICATION).
   - Fase 1: Generar data-model.md, contracts/, quickstart.md.
   - Volver a evaluar la Comprobación de la constitución después del diseño.

## Hooks obligatorios posteriores a la ejecución

**DEBES completar esta sección antes de informar al usuario que el proceso terminó.**

Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si no existe o no hay hooks registrados bajo `hooks.after_plan`, pasa al Informe de finalización.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_plan`.
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

El comando termina después del diseño de la Fase 1. Informa la rama, la ruta de IMPL_PLAN y los artefactos generados.

## Fases

### Fase 0: Esquema e investigación

1. **Extraer incógnitas del Contexto técnico** anterior:
   - Por cada NEEDS CLARIFICATION → tarea de investigación.
   - Por cada dependencia → tarea de mejores prácticas.
   - Por cada integración → tarea de patrones.

2. **Generar y despachar agentes de investigación**:

   ```text
   Para cada incógnita del Contexto técnico:
     Tarea: "Investigar {unknown} para {feature context}"
   Para cada elección tecnológica:
     Tarea: "Encontrar mejores prácticas para {tech} en {domain}"
   ```

3. **Consolidar hallazgos** en `research.md` usando el formato:
   - Decisión: [qué se eligió]
   - Justificación: [por qué se eligió]
   - Alternativas consideradas: [qué más se evaluó]

**Salida**: research.md con todos los NEEDS CLARIFICATION resueltos.

### Fase 1: Diseño y contratos

**Prerrequisitos:** `research.md` completo.

1. **Extraer entidades de la especificación de la funcionalidad** → `data-model.md`:
   - Nombre de entidad, campos, relaciones.
   - Reglas de validación derivadas de los requisitos.
   - Transiciones de estado si aplican.

2. **Definir contratos de interfaz** (si el proyecto tiene interfaces externas) → `/contracts/`:
   - Identifica qué interfaces expone el proyecto a usuarios u otros sistemas.
   - Documenta el formato de contrato adecuado para el tipo de proyecto.
   - Ejemplos: APIs públicas para librerías, esquemas de comandos para herramientas CLI, endpoints para servicios web, gramáticas para parsers, contratos de UI para aplicaciones.
   - Omite este paso si el proyecto es puramente interno (scripts de build, herramientas de un solo uso, etc.).

3. **Crear guía de validación quickstart** → `quickstart.md`:
   - Documenta escenarios de validación ejecutables que demuestren que la funcionalidad funciona de extremo a extremo.
   - Incluye prerrequisitos, comandos de configuración, comandos de prueba/ejecución y resultados esperados.
   - Usa vínculos o referencias a los contratos y detalles del modelo de datos en lugar de duplicarlos.
   - No incluyas código completo de implementación, cuerpos completos de modelos/servicios/controladores, migraciones ni suites de pruebas completas.
   - Mantén este artefacto como guía de validación/ejecución; los detalles de implementación corresponden a `tasks.md` y a la fase de implementación.

**Salida**: data-model.md, /contracts/*, quickstart.md.

## Reglas clave

- Usa rutas absolutas para operaciones del sistema de archivos; usa rutas relativas al proyecto para referencias dentro de la documentación.
- Genera ERROR ante fallos de gates o aclaraciones sin resolver.

## Finalizado cuando

- [ ] Se ejecutó el flujo del plan y se generaron los artefactos de diseño.
- [ ] Los hooks de extensiones se despacharon u omitieron conforme a las reglas de la sección Hooks obligatorios posteriores a la ejecución.
- [ ] Se informó al usuario la finalización con la rama, la ruta del plan y los artefactos generados.
