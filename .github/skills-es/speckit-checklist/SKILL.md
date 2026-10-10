---
name: "speckit-checklist"
description: "Generar una checklist personalizada para la funcionalidad actual basándose en los requisitos del usuario."
compatibility: "Requiere la estructura de proyecto de spec-kit con el directorio .specify/"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/checklist.md"
---


## Propósito de la checklist: "Pruebas unitarias para requisitos en lenguaje natural"

**CONCEPTO CRÍTICO**: Las checklists son **PRUEBAS UNITARIAS PARA LA REDACCIÓN DE REQUISITOS**: validan la calidad, claridad y completitud de los requisitos en un dominio determinado.

**NO son para verificación/pruebas de implementación**:

- ❌ NO "Verifica que el botón responda correctamente al clic"
- ❌ NO "Prueba que funcione el manejo de errores"
- ❌ NO "Confirma que la API devuelva 200"
- ❌ NO comprobar si el código/la implementación coincide con la especificación

**SÍ son para validar la calidad de los requisitos**:

- ✅ "¿Están definidos los requisitos de jerarquía visual para todos los tipos de tarjeta?" (completitud)
- ✅ "¿Se cuantifica 'visualización destacada' con tamaños/posiciones específicos?" (claridad)
- ✅ "¿Son consistentes los requisitos de estado hover en todos los elementos interactivos?" (consistencia)
- ✅ "¿Están definidos los requisitos de accesibilidad para navegación con teclado?" (cobertura)
- ✅ "¿La especificación define qué ocurre cuando no se puede cargar la imagen del logotipo?" (casos límite)

**Metáfora**: Si tu especificación fuera código escrito en lenguaje natural, la checklist sería su suite de pruebas unitarias. Estás comprobando si los requisitos están bien redactados, completos, sin ambigüedades y listos para implementación; **NO** si la implementación funciona.

**Propiedad y ciclo de vida de las casillas**:

- Las checklists personalizadas generadas por este comando son artefactos de revisión de calidad de requisitos cuyo responsable es el revisor.
- `[x]` significa que el revisor determinó que se cumple el criterio de calidad de requisitos.
- `[x]` **NO** significa que el trabajo de implementación esté terminado.
- Este comando genera o agrega elementos a la checklist; **NO DEBE** marcar como `[x]` los elementos generados.
- Un agente puede ayudar a evaluar elementos únicamente cuando el revisor lo solicite explícitamente.
- `checklists/requirements.md` es una checklist incorporada independiente para calidad de especificaciones mantenida por `/speckit-specify` y `/speckit-clarify`; no consideres que esa excepción aplica a las checklists personalizadas generadas aquí.

## Entrada del usuario

```text
$ARGUMENTS
```

**DEBES** considerar la entrada del usuario antes de continuar (si no está vacía).

## Comprobaciones previas a la ejecución

**Comprobar hooks de extensiones (antes de generar la checklist)**:
- Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.before_checklist`.
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

    Espera el resultado del comando del hook antes de continuar con los Pasos de ejecución.
    ```
    Después de emitir el bloque anterior, **DEBES** invocar realmente el hook y esperar a que termine antes de continuar. Ejecútalo del mismo modo en que ejecutarías tú mismo el comando en este agente/sesión (la invocación puede diferir del id literal `{command}` mostrado arriba; por ejemplo, un agente en modo skills lo ejecuta como `/skill:speckit-...` o `$speckit-...`). Emitir el bloque por sí solo no ejecuta el hook.
- Si no hay hooks registrados o `.specify/extensions.yml` no existe, omite silenciosamente esta sección.

## Pasos de ejecución

1. **Configuración**: Ejecuta `.specify/scripts/powershell/check-prerequisites.ps1 -Json -Template checklist-template` desde la raíz del repositorio y analiza el JSON para obtener FEATURE_DIR, la lista AVAILABLE_DOCS y TEMPLATE_CONTENT.
   - Todas las rutas de archivo deben ser absolutas.
   - Para comillas simples en argumentos como "I'm Groot", usa sintaxis de escape: por ejemplo, 'I'\''m Groot' (o comillas dobles si es posible: "I'm Groot").

2. **SI EXISTE**: Carga `.specify/memory/constitution.md` para obtener los principios del proyecto y las restricciones de gobierno.

3. **Aclarar la intención (dinámico)**: Deriva hasta TRES preguntas de aclaración contextuales iniciales (sin catálogo prefabricado). **DEBEN**:
   - Generarse a partir de la redacción del usuario + señales extraídas de spec/plan/tasks.
   - Preguntar únicamente por información que cambie materialmente el contenido de la checklist.
   - Omitirse individualmente si ya están inequívocamente resueltas en `$ARGUMENTS`.
   - Priorizar precisión sobre amplitud.

   Algoritmo de generación:
   1. Extrae señales: palabras clave del dominio de la funcionalidad (por ejemplo, auth, latencia, UX, API), indicadores de riesgo ("crítico", "debe", "cumplimiento"), indicios de stakeholders ("QA", "revisión", "equipo de seguridad") y entregables explícitos ("a11y", "rollback", "contratos").
   2. Agrupa las señales en áreas de enfoque candidatas (máx. 4), ordenadas por relevancia.
   3. Identifica audiencia y momento probables (autor, revisor, QA, release) si no son explícitos.
   4. Detecta dimensiones faltantes: amplitud del alcance, profundidad/rigor, énfasis en riesgos, límites de exclusión y criterios de aceptación medibles.
   5. Formula preguntas elegidas de estos arquetipos:
      - Refinamiento de alcance (por ejemplo, "¿Debe incluir puntos de integración con X e Y o limitarse a la corrección del módulo local?")
      - Priorización de riesgos (por ejemplo, "¿Cuáles de estas áreas de riesgo potencial deberían tener comprobaciones obligatorias como gate?")
      - Calibración de profundidad (por ejemplo, "¿Es una lista ligera de sanity check previa al commit o un gate formal de release?")
      - Enfoque de audiencia (por ejemplo, "¿La usará solo el autor o también sus pares durante la revisión del PR?")
      - Exclusión de límites (por ejemplo, "¿Debemos excluir explícitamente los elementos de ajuste de rendimiento en esta ronda?")
      - Brecha de clase de escenario (por ejemplo, "No se detectaron flujos de recuperación: ¿los caminos de rollback/fallo parcial están dentro del alcance?")

   Reglas de formato de las preguntas:
   - Si presentas opciones, genera una tabla compacta con columnas: Opción | Candidato | Por qué importa.
   - Limita a un máximo de opciones A–E; omite la tabla si una respuesta libre resulta más clara.
   - Nunca pidas al usuario que repita algo que ya dijo.
   - Evita categorías especulativas (no inventes). Si hay incertidumbre, pregunta explícitamente: "Confirma si X está dentro del alcance."

   Valores predeterminados cuando la interacción no sea posible:
   - Profundidad: Estándar.
   - Audiencia: Revisor (PR) si está relacionado con código; Autor en caso contrario.
   - Enfoque: Los 2 clústeres de mayor relevancia.

   Muestra las preguntas (etiquétalas Q1/Q2/Q3). Después de las respuestas: si ≥2 clases de escenario (Alternativo / Excepción / Recuperación / Dominio no funcional) siguen sin estar claras, **PUEDES** hacer hasta DOS preguntas de seguimiento adicionales y dirigidas (Q4/Q5), cada una con una justificación de una línea (por ejemplo, "Riesgo de ruta de recuperación sin resolver"). No superes cinco preguntas en total. Omite esta ampliación si el usuario rechaza explícitamente más preguntas.

4. **Comprender la solicitud del usuario**: Combina `$ARGUMENTS` + las respuestas de aclaración:
   - Deriva el tema de la checklist (por ejemplo, seguridad, revisión, despliegue, UX).
   - Consolida los elementos imprescindibles mencionados explícitamente por el usuario.
   - Asocia las selecciones de enfoque con la estructura de categorías.
   - Infiere cualquier contexto faltante a partir de spec/plan/tasks (**NO** inventes).

5. **Cargar el contexto de la funcionalidad**: Lee desde FEATURE_DIR:
   - spec.md: Requisitos y alcance de la funcionalidad.
   - plan.md (si existe): Detalles técnicos, dependencias.
   - tasks.md (si existe): Tareas de implementación.

   **Estrategia de carga de contexto**:
   - Carga únicamente las partes necesarias relevantes para las áreas de enfoque activas (evita volcar archivos completos).
   - Prefiere resumir secciones largas en viñetas concisas de escenarios/requisitos.
   - Usa divulgación progresiva: realiza recuperaciones adicionales solo si se detectan brechas.
   - Si los documentos fuente son grandes, genera elementos de resumen intermedios en lugar de incrustar texto sin procesar.

6. **Generar la checklist**: Usa TEMPLATE_CONTENT como plantilla estructural y crea "Pruebas unitarias para requisitos":
   - Crea el directorio `FEATURE_DIR/checklists/` si no existe.
   - Genera un nombre de archivo único para la checklist:
     - Usa un nombre corto y descriptivo basado en el dominio (por ejemplo, `ux.md`, `api.md`, `security.md`).
     - Formato: `[domain].md`.
   - Comportamiento de manejo del archivo:
     - Si el archivo **NO** existe: crea un archivo nuevo y numera los elementos comenzando en CHK001.
     - Si el archivo existe: agrega los nuevos elementos al archivo existente, continuando desde el último ID CHK (por ejemplo, si el último es CHK015, comienza en CHK016).
   - Nunca elimines ni reemplaces contenido existente de la checklist; consérvalo siempre y agrega al final.
   - Deja todos los elementos recién generados sin marcar (`[ ]`); el estado de la casilla pertenece al revisor.

   **PRINCIPIO CENTRAL: Probar los requisitos, no la implementación**:
   Cada elemento de la checklist **DEBE** evaluar LOS PROPIOS REQUISITOS respecto de:
   - **Completitud**: ¿Están presentes todos los requisitos necesarios?
   - **Claridad**: ¿Los requisitos son inequívocos y específicos?
   - **Consistencia**: ¿Los requisitos están alineados entre sí?
   - **Medibilidad**: ¿Los requisitos pueden verificarse objetivamente?
   - **Cobertura**: ¿Están contemplados todos los escenarios/casos límite?

   **Estructura de categorías**: Agrupa los elementos por dimensiones de calidad de requisitos:
   - **Completitud de requisitos** (¿Están documentados todos los requisitos necesarios?)
   - **Claridad de requisitos** (¿Los requisitos son específicos e inequívocos?)
   - **Consistencia de requisitos** (¿Los requisitos están alineados sin conflictos?)
   - **Calidad de los criterios de aceptación** (¿Los criterios de éxito son medibles?)
   - **Cobertura de escenarios** (¿Se contemplan todos los flujos/casos?)
   - **Cobertura de casos límite** (¿Están definidas las condiciones de frontera?)
   - **Requisitos no funcionales** (Rendimiento, Seguridad, Accesibilidad, etc.; ¿están especificados?)
   - **Dependencias y supuestos** (¿Están documentados y validados?)
   - **Ambigüedades y conflictos** (¿Qué necesita aclaración?)

   **CÓMO REDACTAR LOS ELEMENTOS DE LA CHECKLIST: "Pruebas unitarias para requisitos en lenguaje natural"**:

   ❌ **INCORRECTO** (prueba la implementación):
   - "Verifica que la página de inicio muestre 3 tarjetas de episodios"
   - "Prueba que los estados hover funcionen en escritorio"
   - "Confirma que hacer clic en el logotipo navegue al inicio"

   ✅ **CORRECTO** (prueba la calidad de los requisitos):
   - "¿Se especifican la cantidad exacta y la distribución de los episodios destacados?" [Completitud]
   - "¿Se cuantifica 'visualización destacada' con tamaños/posiciones específicos?" [Claridad]
   - "¿Son consistentes los requisitos de estado hover en todos los elementos interactivos?" [Consistencia]
   - "¿Están definidos los requisitos de navegación con teclado para toda la UI interactiva?" [Cobertura]
   - "¿Se especifica el comportamiento alternativo cuando no se puede cargar la imagen del logotipo?" [Casos límite]
   - "¿Están definidos los estados de carga para los datos asíncronos de episodios?" [Completitud]
   - "¿La especificación define la jerarquía visual para elementos de UI que compiten entre sí?" [Claridad]

   **ESTRUCTURA DEL ELEMENTO**:
   Cada elemento debería seguir este patrón:
   - Formato de pregunta sobre la calidad del requisito.
   - Centrarse en lo que ESTÁ ESCRITO (o no está escrito) en la especificación/plan.
   - Incluir la dimensión de calidad entre corchetes [Completitud/Claridad/Consistencia/etc.].
   - Referenciar la sección de la especificación `[Spec §X.Y]` al comprobar requisitos existentes.
   - Usar el marcador `[Gap]` al comprobar requisitos faltantes.

   **EJEMPLOS POR DIMENSIÓN DE CALIDAD**:

   Completitud:
   - "¿Están definidos los requisitos de manejo de errores para todos los modos de fallo de la API? [Gap]"
   - "¿Están especificados los requisitos de accesibilidad para todos los elementos interactivos? [Completitud]"
   - "¿Están definidos los requisitos de breakpoints móviles para layouts responsivos? [Gap]"

   Claridad:
   - "¿Se cuantifica 'carga rápida' con umbrales de tiempo específicos? [Claridad, Spec §NFR-2]"
   - "¿Están definidos explícitamente los criterios de selección de 'episodios relacionados'? [Claridad, Spec §FR-5]"
   - "¿Se define 'destacado' mediante propiedades visuales medibles? [Ambigüedad, Spec §FR-4]"

   Consistencia:
   - "¿Los requisitos de navegación están alineados en todas las páginas? [Consistencia, Spec §FR-10]"
   - "¿Son consistentes los requisitos del componente tarjeta entre las páginas de inicio y detalle? [Consistencia]"

   Cobertura:
   - "¿Hay requisitos definidos para escenarios de estado vacío (sin episodios)? [Cobertura, Caso límite]"
   - "¿Se contemplan escenarios de interacción concurrente de usuarios? [Cobertura, Gap]"
   - "¿Se especifican requisitos para fallos parciales en la carga de datos? [Cobertura, Flujo de excepción]"

   Medibilidad:
   - "¿Son medibles/comprobables los requisitos de jerarquía visual? [Criterios de aceptación, Spec §FR-1]"
   - "¿Puede verificarse objetivamente el 'peso visual equilibrado'? [Medibilidad, Spec §FR-2]"

   **Clasificación y cobertura de escenarios** (enfoque en calidad de requisitos):
   - Comprueba si existen requisitos para escenarios: Primario, Alternativo, Excepción/Error, Recuperación y No funcional.
   - Para cada clase de escenario, pregunta: "¿Los requisitos de [tipo de escenario] son completos, claros y consistentes?"
   - Si falta una clase de escenario: "¿Los requisitos de [tipo de escenario] se excluyeron intencionalmente o faltan? [Gap]"
   - Incluye resiliencia/rollback cuando haya mutación de estado: "¿Están definidos los requisitos de rollback para fallos de migración? [Gap]"

   **Requisitos de trazabilidad**:
   - MÍNIMO: ≥80% de los elementos **DEBEN** incluir al menos una referencia de trazabilidad.
   - Cada elemento debería referenciar una sección de la especificación `[Spec §X.Y]` o usar marcadores: `[Gap]`, `[Ambiguity]`, `[Conflict]`, `[Assumption]`.
   - Si no existe un sistema de IDs: "¿Está establecido un esquema de IDs para requisitos y criterios de aceptación? [Trazabilidad]"

   **Hacer visibles y resolver problemas** (problemas de calidad de requisitos):
   Haz preguntas sobre los propios requisitos:
   - Ambigüedades: "¿Se cuantifica el término 'rápido' con métricas específicas? [Ambigüedad, Spec §NFR-1]"
   - Conflictos: "¿Los requisitos de navegación entran en conflicto entre §FR-10 y §FR-10a? [Conflicto]"
   - Supuestos: "¿Está validado el supuesto de 'API de podcast siempre disponible'? [Supuesto]"
   - Dependencias: "¿Están documentados los requisitos de la API externa de podcast? [Dependencia, Gap]"
   - Definiciones faltantes: "¿Está definida la 'jerarquía visual' mediante criterios medibles? [Gap]"

   **Consolidación de contenido**:
   - Límite flexible: si los elementos candidatos sin filtrar > 40, prioriza por riesgo/impacto.
   - Fusiona casi duplicados que comprueben el mismo aspecto del requisito.
   - Si hay >5 casos límite de bajo impacto, crea un único elemento: "¿Los casos límite X, Y, Z están contemplados en los requisitos? [Cobertura]"

   **🚫 ABSOLUTAMENTE PROHIBIDO**: Esto lo convertiría en una prueba de implementación, no de requisitos:
   - ❌ Cualquier elemento que empiece con "Verifica", "Prueba", "Confirma", "Comprueba" + comportamiento de implementación.
   - ❌ Referencias a ejecución de código, acciones del usuario o comportamiento del sistema.
   - ❌ "Se muestra correctamente", "funciona correctamente", "funciona según lo esperado".
   - ❌ "Hacer clic", "navegar", "renderizar", "cargar", "ejecutar".
   - ❌ Casos de prueba, planes de prueba, procedimientos de QA.
   - ❌ Detalles de implementación (frameworks, APIs, algoritmos).

   **✅ PATRONES OBLIGATORIOS**: Estos evalúan la calidad de los requisitos:
   - ✅ "¿Están [tipo de requisito] definidos/especificados/documentados para [escenario]?"
   - ✅ "¿Se cuantifica/aclara [término vago] mediante criterios específicos?"
   - ✅ "¿Los requisitos son consistentes entre [sección A] y [sección B]?"
   - ✅ "¿Puede [requisito] medirse/verificarse objetivamente?"
   - ✅ "¿Los [casos límite/escenarios] están contemplados en los requisitos?"
   - ✅ "¿La especificación define [aspecto faltante]?"

7. **Referencia de estructura**: Genera la checklist siguiendo la plantilla canónica de `.specify/templates/checklist-template.md` para el título, sección meta, encabezados de categorías, nota de propiedad, sección de notas y formato de IDs. Si la plantilla no está disponible, usa: título H1, líneas meta de propósito/creación, una nota de propiedad explicando que `[x]` significa aprobación del revisor sobre la calidad de los requisitos, secciones de categoría `##` que contengan líneas `- [ ] CHK### <elemento del requisito>` con IDs incrementales globales comenzando en CHK001, y notas que indiquen que `/speckit-implement` lee el estado de la checklist pero no modifica los marcadores.

8. **Informe**: Muestra la ruta completa del archivo de checklist, el número de elementos y resume si la ejecución creó un archivo nuevo o agregó contenido a uno existente. Resume:
   - Áreas de enfoque seleccionadas.
   - Nivel de profundidad.
   - Actor/momento.
   - Cualquier elemento imprescindible especificado explícitamente por el usuario que se haya incorporado.

**Importante**: Cada invocación del comando `/speckit-checklist` usa un nombre de archivo corto y descriptivo para la checklist y crea un archivo nuevo o agrega a uno existente. Esto permite:

- Múltiples checklists de distintos tipos (por ejemplo, `ux.md`, `test.md`, `security.md`).
- Nombres de archivo simples y fáciles de recordar que indiquen el propósito de la checklist.
- Identificación y navegación sencillas en la carpeta `checklists/`.

Para evitar desorden, usa tipos descriptivos y elimina las checklists obsoletas cuando corresponda.

## Tipos de checklist de ejemplo y elementos de muestra

**Calidad de requisitos de UX:** `ux.md`

Elementos de muestra (evalúan los requisitos, **NO** la implementación):

- "¿Están definidos los requisitos de jerarquía visual mediante criterios medibles? [Claridad, Spec §FR-1]"
- "¿Se especifican explícitamente la cantidad y posición de los elementos de UI? [Completitud, Spec §FR-1]"
- "¿Se definen de forma consistente los requisitos de estados de interacción (hover, focus, active)? [Consistencia]"
- "¿Están especificados los requisitos de accesibilidad para todos los elementos interactivos? [Cobertura, Gap]"
- "¿Está definido el comportamiento alternativo cuando no se pueden cargar imágenes? [Caso límite, Gap]"
- "¿Puede medirse objetivamente 'visualización destacada'? [Medibilidad, Spec §FR-4]"

**Calidad de requisitos de API:** `api.md`

Elementos de muestra:

- "¿Están especificados los formatos de respuesta de error para todos los escenarios de fallo? [Completitud]"
- "¿Están cuantificados los requisitos de rate limiting mediante umbrales específicos? [Claridad]"
- "¿Son consistentes los requisitos de autenticación en todos los endpoints? [Consistencia]"
- "¿Están definidos los requisitos de reintento/timeout para dependencias externas? [Cobertura, Gap]"
- "¿Está documentada la estrategia de versionado en los requisitos? [Gap]"

**Calidad de requisitos de rendimiento:** `performance.md`

Elementos de muestra:

- "¿Están cuantificados los requisitos de rendimiento mediante métricas específicas? [Claridad]"
- "¿Están definidos los objetivos de rendimiento para todos los recorridos críticos del usuario? [Cobertura]"
- "¿Están especificados los requisitos de rendimiento bajo distintas condiciones de carga? [Completitud]"
- "¿Pueden medirse objetivamente los requisitos de rendimiento? [Medibilidad]"
- "¿Están definidos los requisitos de degradación para escenarios de alta carga? [Caso límite, Gap]"

**Calidad de requisitos de seguridad:** `security.md`

Elementos de muestra:

- "¿Están especificados los requisitos de autenticación para todos los recursos protegidos? [Cobertura]"
- "¿Están definidos los requisitos de protección de datos para información sensible? [Completitud]"
- "¿Está documentado el modelo de amenazas y los requisitos están alineados con él? [Trazabilidad]"
- "¿Son consistentes los requisitos de seguridad con las obligaciones de cumplimiento? [Consistencia]"
- "¿Están definidos los requisitos de respuesta ante fallos/brechas de seguridad? [Gap, Flujo de excepción]"

## Anti-ejemplos: Qué NO hacer

**❌ INCORRECTO: Estos prueban la implementación, no los requisitos:**

```markdown
- [ ] CHK001 - Verifica que la página de inicio muestre 3 tarjetas de episodios [Spec §FR-001]
- [ ] CHK002 - Prueba que los estados hover funcionen correctamente en escritorio [Spec §FR-003]
- [ ] CHK003 - Confirma que hacer clic en el logotipo navegue a la página de inicio [Spec §FR-010]
- [ ] CHK004 - Comprueba que la sección de episodios relacionados muestre 3-5 elementos [Spec §FR-005]
```

**✅ CORRECTO: Estos prueban la calidad de los requisitos:**

```markdown
- [ ] CHK001 - ¿Se especifican explícitamente la cantidad y distribución de episodios destacados? [Completitud, Spec §FR-001]
- [ ] CHK002 - ¿Se definen de forma consistente los requisitos de estado hover para todos los elementos interactivos? [Consistencia, Spec §FR-003]
- [ ] CHK003 - ¿Son claros los requisitos de navegación para todos los elementos de marca clicables? [Claridad, Spec §FR-010]
- [ ] CHK004 - ¿Están documentados los criterios de selección de episodios relacionados? [Gap, Spec §FR-005]
- [ ] CHK005 - ¿Están definidos los requisitos de estado de carga para datos asíncronos de episodios? [Gap]
- [ ] CHK006 - ¿Pueden medirse objetivamente los requisitos de "jerarquía visual"? [Medibilidad, Spec §FR-001]
```

**Diferencias clave:**

- Incorrecto: Comprueba si el sistema funciona correctamente.
- Correcto: Comprueba si los requisitos están redactados correctamente.
- Incorrecto: Verificación del comportamiento.
- Correcto: Validación de la calidad de los requisitos.
- Incorrecto: "¿Hace X?"
- Correcto: "¿X está claramente especificado?"

## Comprobaciones posteriores a la ejecución

**Comprobar hooks de extensiones (después de generar la checklist)**:
Comprueba si `.specify/extensions.yml` existe en la raíz del proyecto.
- Si existe, léelo y busca entradas bajo la clave `hooks.after_checklist`.
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
