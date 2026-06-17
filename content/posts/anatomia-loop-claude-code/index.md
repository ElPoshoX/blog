---
title: "Anatomía de un Loop en Claude Code: Los Bloques, el Diseño y las Métricas que Importan"
weight: 1
tags: ["claude-code", "loop-engineering", "automation", "sub-agents", "devops", "metrics"]
date: "2026-06-12T10:00:00+00:00"
showDateUpdated: false
cascade:
  showDate: false
  showAuthor: false
  invertPagination: true
---

# Anatomía de un Loop en Claude Code: Los Bloques, el Diseño y las Métricas que Importan

{{< lead >}}
Los cinco bloques de un loop desarmados, un loop real paso a paso, las métricas que deberías trackear, y por qué el sub-agent reviewer es el componente más importante de todo el sistema.
{{< /lead >}}

Este post es el companion técnico de [El Loop Que Nadie Testea]. Si quieres el argumento, empieza por ahí.

Llevo meses armando loops en Claude Code para automatizar tareas en mis repos. Algunos jalan bien. Otros quemaron tokens sin producir nada útil. Aquí desarmo cada pieza.

Una cosa antes de arrancar: todo lo que vas a ver aquí funciona con tu suscripción de Claude Code. Cero costo extra. Sin APIs de terceros de paga, sin servicios cloud adicionales. Solo Claude Code y lo que ya trae.

## Los cinco bloques de un loop, desarmados

Un loop en Claude Code no es una cosa monolítica. Son cinco bloques que puedes combinar. Entender cada uno por separado te da control real sobre el comportamiento del sistema.

{{< mermaid >}}
graph TD
    A[Automations] -->|dispara| B[Execution Engine]
    B -->|corre en| C[Worktrees]
    B -->|sigue reglas de| D[Skills]
    B -->|se conecta via| E[Connectors / MCP]
    B -->|delega a| F[Sub-agents]
    B -->|persiste estado en| G[State Files]
    F -->|maker + reviewer| B
{{< /mermaid >}}

### Bloque 1: Automations

Las automations son lo que dispara tu loop. El trigger. Sin esto, no hay ciclo.

La herramienta más directa es `/loop`. La usas para tareas recurrentes que el agente ejecuta con auto-pacing.

Ejemplo real: un `/loop` para triage matutino de issues.

```
/loop 30m

Revisa los issues abiertos en el repo elposhox/tekal-api.
Para cada issue nuevo (creado en las últimas 24h):
1. Lee el título y body
2. Clasifica como: bug, feature-request, question, o needs-info
3. Agrega el label correspondiente
4. Si es bug con error message claro y afecta un solo archivo, agrega label "autofix-candidate"
5. Escribe un resumen en AGENTS.md con: issue number, clasificación, y si es candidato a autofix

Detente después de procesar todos los issues nuevos o después de 3 iteraciones sin encontrar issues nuevos.
```

Eso corre cada 30 minutos. Sin intervención. El agente revisa, clasifica, y te deja un registro.

También tienes `CronCreate` para schedules más específicos y `ScheduleWakeup` para delays puntuales. Pero `/loop` es el que vas a usar el 80% del tiempo.

### Bloque 2: Worktrees

Cuando tu loop modifica código, necesitas aislamiento. Si el agente edita archivos en tu branch de trabajo, te va a hacer un desmadre. Los worktrees resuelven esto.

En Claude Code configuras aislamiento de dos formas:

Para sub-agents que lanzas con el Agent tool:

```json
{
  "description": "Fix issue in isolation",
  "prompt": "Fix the bug described in issue #42...",
  "isolation": "worktree"
}
```

El agente recibe su propio worktree. Trabaja ahí. Tu branch principal no se toca hasta que tú decides mergear.

Para agent presets en `.claude/agents/`, agregas la directiva en el prompt:

```markdown
# maker.md
You MUST work in an isolated worktree for all code changes.
Use the EnterWorktree tool at the start of every task.
Never modify files in the main working directory directly.
```

Si tu loop toca código y no usa worktrees, es cuestión de tiempo para que te rompa algo.

### Bloque 3: Skills

Los skills son las instrucciones que le dan criterio al agente. La diferencia entre un SKILL.md efectivo y uno que el agente ignora es la calidad de su definición.

**SKILL.md que el agente ignora:**

```markdown
# triage.md
Revisa issues y clasifícalos apropiadamente.
Usa buen criterio para determinar la prioridad.
```

Eso no dice nada. "Buen criterio" no es una instrucción.

**SKILL.md que el agente sigue:**

```markdown
# triage.md
## Criterios de clasificación

Un issue es un BUG si:
- Tiene un error message o stack trace
- Describe comportamiento que funcionaba antes y dejó de funcionar
- Incluye pasos para reproducir

Un issue es AUTOFIX-CANDIDATE si cumple TODOS estos:
- Es un bug con error message claro
- El error apunta a un solo archivo (grep del filename en el stack trace)
- No requiere cambios de schema de base de datos
- No requiere cambios en más de 2 archivos

Un issue es NEEDS-INFO si:
- No tiene pasos para reproducir
- El reporte es ambiguo ("no funciona" sin detalles)
- No incluye versión o environment

## Qué NO hacer
- No cambies labels de issues que ya tienen clasificación
- No comentes en issues que ya tienen respuesta del maintainer
- No intentes fixes de issues que requieren cambios de API pública
```

**Qué va en CLAUDE.md vs qué va en skills separados:**

| En CLAUDE.md | En skills separados |
|---|---|
| Convenciones globales del proyecto | Instrucciones de tareas específicas |
| Reglas de estilo y linting | Criterios de clasificación |
| Paths importantes | Workflows paso a paso |
| Cosas que aplican a TODA interacción | Cosas que aplican solo a UN tipo de tarea |

Tu CLAUDE.md es lo que el agente lee siempre. Los skills los lee cuando los necesita. Si metes todo en CLAUDE.md, el contexto se infla y el agente prioriza tomar decisiones no tan buenas con información que puede o no ser relevante en ese punto del tiempo.

### Bloque 4: Connectors (MCP Servers)

Los connectors son MCP servers que le dan al agente acceso a servicios externos. GitHub, Linear, Jira, tu base de datos, lo que necesites.

Ejemplo: un MCP server para GitHub que el agente usa dentro del loop.

En tu `.claude/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

Con eso, tu agente puede listar issues, leer PRs, crear comments, abrir PRs. Todo desde dentro del loop sin salir a bash a hacer `gh` calls.

Para Linear es similar:

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-linear"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    }
  }
}
```

Lo importante: cada MCP server que agregas es un conjunto de tools que el agente puede usar. Más tools = más contexto = más tokens por iteración. No agregues servers que no necesitas para el loop.

### Bloque 5: Sub-agents

Un loop serio no es un solo agente haciendo todo. Son sub-agents con roles distintos.

La configuración vive en `.claude/agents/`. Cada archivo es un agent preset con sus propias instrucciones.

**El maker** (`maker.md`):

```markdown
# Maker Agent

## Rol
Implementar fixes para issues clasificados como autofix-candidate.

## Instrucciones
1. Lee el issue completo (título, body, comments)
2. Identifica el archivo afectado desde el stack trace
3. Entra a un worktree aislado
4. Implementa el fix más simple que resuelva el problema
5. Corre los tests del proyecto
6. Si los tests pasan, commitea con mensaje: "fix: resolve #ISSUE_NUMBER - DESCRIPCION_CORTA"
7. Si los tests fallan después de 2 intentos de fix, abandona y registra en AGENTS.md

## Restricciones
- Máximo 2 archivos modificados por fix
- No cambies tests existentes para que pasen (eso es trampa)
- No agregues dependencias nuevas
- Máximo 3 iteraciones por issue
```

**El reviewer** (`reviewer.md`) lo detallo en su propia sección más abajo porque es el componente más importante.

### Bloque 6: State (Progress Files)

Sin estado persistente, tu loop no tiene memoria entre runs. Cada ejecución empieza de cero. El progress file resuelve esto.

Diseño de un `AGENTS.md` que funciona:

```markdown
# Loop State - Issue Triage & Autofix

## Last Run
- Date: 2026-06-12T08:30:00Z
- Issues processed: 5
- Fixes attempted: 2
- PRs opened: 1

## Issue Log

| Issue | Status | Classification | Fix Attempted | Result | PR |
|-------|--------|---------------|---------------|--------|----|
| #142 | done | bug/autofix-candidate | yes | fix merged | #148 |
| #143 | done | feature-request | no | N/A | N/A |
| #144 | done | bug/autofix-candidate | yes | tests failed, abandoned | N/A |
| #145 | done | needs-info | no | commented asking for repro | N/A |
| #146 | done | question | no | N/A | N/A |

## Metrics (rolling 7 days)
- PRs opened: 4
- PRs merged without changes: 3
- PRs that needed manual fixes: 1
- Issues abandoned: 2
- Total iterations: 23
```

El agente lee esto al inicio de cada run. Sabe qué ya procesó. Sabe qué falló antes. No repite trabajo.

**Formato importante:** usa Markdown con tablas. El agente lo parsea bien y tú lo lees sin herramientas especiales. JSON funciona también pero es más difícil de revisar a ojo.

## Diseño de un loop concreto, paso a paso

Loop real que corrí sobre un Kubernetes operator propio (tekal-waf-sync). El repo tiene un spec pendiente: agregar integration tests con Moto y una capa de validación al cliente de WAF. 7 tasks definidas en un archivo de tasks. El loop las toma una por una, implementa en worktree aislado, corre tests y lint, pasa al reviewer, y abre PR si todo pasa.

{{< mermaid >}}
sequenceDiagram
    participant Orch as Loop Principal
    participant State as AGENTS.md
    participant Spec as tasks.md
    participant Maker as Maker Agent
    participant Reviewer as Reviewer Agent
    participant GH as GitHub (MCP)

    Orch->>State: Lee estado previo
    Orch->>Spec: Lee tasks pendientes
    Orch->>Orch: Identifica siguiente task no completada

    Note over Orch,GH: Se repite para cada task pendiente

    Orch->>Maker: Implementa task N
    Maker->>Maker: Entra worktree aislado
    Maker->>Maker: Implementa según spec
    Maker->>Maker: Corre tests + lint

    alt Tests y lint pasan
        Maker->>Reviewer: Review implementación de task N
        alt Reviewer aprueba
            Reviewer->>GH: Abre PR
            Reviewer->>State: Registra PR + task completada
        else Reviewer rechaza
            Reviewer->>Maker: Feedback específico
            Maker->>Maker: Segundo intento
            Maker->>Reviewer: Re-review
        end
    else Tests fallan 3 veces
        Maker->>State: Registra abandono con razón
    end
{{< /mermaid >}}

### Paso 1: El prompt del loop

```
/loop

Implementa las tasks pendientes del spec waf-integration-tests.

1. Lee AGENTS.md para saber qué tasks ya completaste
2. Lee openspec/changes/waf-integration-tests/tasks.md para obtener la lista completa
3. Identifica la siguiente task cuyas dependencias estén satisfechas
4. Para cada task:
   a. Lanza un sub-agent maker con isolation: worktree
   b. El maker implementa según la descripción y acceptance criteria de la task
   c. El maker corre `make test` y `make lint`
   d. Si pasa, lanza el sub-agent reviewer
   e. Si el reviewer aprueba, commitea y abre PR via GitHub MCP
   f. Registra resultado en AGENTS.md
5. Detente cuando: todas las tasks estén completadas, O después de 3 tasks por run, O si llevas más de 45 minutos

Al terminar, actualiza AGENTS.md con métricas del run.
```

### Paso 2: El skill con criterios del spec

En vez de un skill de triage genérico, el loop usa el spec como fuente de verdad. El tasks.md define para cada task:

- Qué archivos crear o modificar
- Acceptance criteria explícito (tests pasan, coverage ≥80%, lint clean)
- Dependencias entre tasks (task 3 depende de task 1, task 5 depende de task 4)

El maker no necesita "decidir qué hacer". Solo necesita implementar lo que el spec dice. La ambigüedad la resuelves al escribir el spec, no al correr el loop.

[!IMPORTANT] 
El uso de spec aquí es totalmente opcional, yo trabajo así porque me permite dos cosas importantes para mí.
-  Entender que se va a crear antes de que se implemente.
- Generar deltas en cada spec (usando OpenSpec) que se vuelven documentación para el proyecto.

### Paso 3: El sub-agent maker que implementa

Para este loop, el maker tiene instrucciones diferentes a un maker de bugfixes:

- **Puede crear archivos nuevos** (el spec lo pide: `internal/waf/validation.go`, `test/integration/`)
- **Puede agregar dependencias** (testcontainers-go para LocalStack)
- **Tiene el spec como input** (no un issue de GitHub, sino una task con acceptance criteria)
- **Máximo 5 iteraciones** (más que un bugfix porque es implementación nueva)

El constraint que no cambia: si tests o lint fallan después de 3 intentos, abandona y registra por qué.

### Paso 4: El sub-agent reviewer

Detallado en su propia sección abajo. Para tasks de spec, el reviewer verifica contra el acceptance criteria definido, no contra "buenas prácticas" genéricas.

### Paso 5: El progress file

```markdown
# Loop State - waf-integration-tests

## Last Run
- Date: 2026-06-16T10:30:00Z
- Tasks attempted: 2
- PRs opened: 2
- Tasks abandoned: 0

## Task Log

| Task | Status | Iterations | Result | PR |
|------|--------|-----------|--------|-----|
| 1. validation.go | done | 2 | merged | #23 |
| 2. client integration | done | 1 | merged | #24 |
| 3. validation tests | in-progress | - | - | - |
| 4. testcontainers setup | pending | - | - | - |
| 5. WAF integration tests | pending | - | - | - |
| 6. reconciler integration | pending | - | - | - |
| 7. CI pipeline | pending | - | - | - |

## Metrics
- PRs opened: 2
- PRs merged sin cambios: 2
- Average iterations per task: 1.5
- Coverage delta: +4.2%
```

Walk-through del primer run:

1. Loop arranca. Lee AGENTS.md: vacío, primer run. Lee tasks.md: 7 tasks, task 1 no tiene dependencias.
2. Lanza maker para task 1: "Crear `internal/waf/validation.go` con ValidateRuleGroupInput y ValidateByteMatchStatement." El maker entra a worktree, implementa las funciones de validación, corre `make test` y `make lint`. Pasa.
3. Lanza reviewer. Verifica: ¿funciones ≤50 líneas? ¿Coverage del nuevo archivo ≥80%? ¿Sigue convenciones del CLAUDE.md? Aprueba.
4. Abre PR #23. Actualiza AGENTS.md.
5. Siguiente task con dependencias satisfechas: task 2 (depende de task 1, que ya está done). Lanza maker. Implementa la integración en `client.go`. Tests pasan. Reviewer aprueba. PR #24.
6. Task 3 depende de task 1 (done). Podría continuar, pero lleva 2 tasks y el budget dice máximo 3. Continúa.
7. Implementa task 3: unit tests table-driven para la capa de validación. 3 iteraciones porque el primer intento no cubría el edge case de InsertHeaders vacío. Reviewer lo rechazó. Segundo intento: ok. PR #25.
8. Ya lleva 3 tasks. Se detiene. Actualiza AGENTS.md con métricas.

## Resultados del loop sobre tekal-waf-sync

Corrí este loop sobre las 7 tasks del spec. Estos son los números reales:

| Métrica | Valor |
|---------|-------|
| Tasks del spec | 8 (7 de implementación + 1 peer review) |
| Tasks completadas por el loop | 8 |
| Commits | 4 |
| Tasks abandonadas | 0 |
| Iteraciones totales | ~12 |
| Tests nuevos escritos | 28 (17 unit + 11 integration) |
| Coverage delta | 100% → 100% (recuperado después de peer review) |
| Errores auto-corregidos | 3 (hugeParam lint, test count mismatch, Capacity *int64) |
| Findings de peer review | 2 (coverage gap en updateShard, containers no compartidos) |
| Intervenciones humanas | 1 (sugerir Floci sobre LocalStack, resultó ser mala idea) |

Lo que funcionó bien:

- Tests table-driven generados correctos al primer intento. 16 casos de test para validación, cubriendo los 6 casos inválidos del spec + boundary values (nombre de exactamente 128 chars, exactamente 1 InsertHeader).
- Los 5 tests del reconciler contra WAF real (moto) pasaron en la primera ejecución sin fixes.
- Lint se mantuvo en 0 issues después de cada fix. El único error (hugeParam, `types.Rule` de 96 bytes) fue detectado y corregido en la misma iteración.
- El patrón testcontainers-go resultó limpio: setup en helper, teardown via `t.Cleanup`.
- El loop respetó sus propias condiciones de parada (3 tasks por run) sin que tuviera que recordárselo.

Lo que falló:

- El research agent hallucinó que Floci soportaba WAFv2 con "35 operaciones". En realidad, Floci regresó `UnknownOperationException`. Costo: ~3 minutos + un Docker pull inútil. La causa fue que yo sugerí Floci como alternativa OSS a LocalStack. El loop confiaba en mi juicio, pero mi juicio estaba mal.
- `go mod tidy` eliminó la dependencia de testcontainers cuando solo se usa detrás de un build tag. Tuvo que re-agregar con `GOFLAGS="-tags=integration"`. Esto es un gotcha conocido de Go modules que el loop no anticipó.
- `ValidateByteMatchStatement` regresa `error` (singular via `errors.Join`), no `[]error`. El test esperaba 3 errores por 3 violaciones pero recibió 2 porque dos violaciones de ByteMatch se unen en un solo error. Corregido en la segunda iteración.

Lo que el loop no pudo hacer y tuve que hacer a mano:

- Task 8 (peer review). El loop identifica correctamente que no debe ejecutar peer review de forma autónoma. Es un proceso que requiere cargar archivos de agentes del vault de Obsidian y tomar decisiones sobre findings. Esto es correcto: el loop PARA MI es un implementador, no un revisor.
- Elegir entre Floci y moto. Cuando Floci falló, el loop recuperó buscando alternativas. Pero la decisión original de probar Floci fue mía (del humano) y resultó ser un camino muerto.
- Decidir si Rule 6 (Capacity > 0) aplica a `UpdateRuleGroupInput`. El loop tomó la decisión correcta (no aplica, Capacity es inmutable después de creación) pero esto requirió razonamiento sobre la semántica del API de AWS, no solo seguir el spec literalmente.

---

## Métricas que deberías trackear

Claude Code no te da un dashboard de métricas. Tú tienes que construir la observabilidad. Estas son las métricas que importan y cómo sacarlas.

### Tasa de aceptación

De N PRs que el loop abre, cuántos mergeas sin cambios.

Si esta tasa es alta (>80%), tu loop está produciendo código de calidad. Si es baja, algo en el maker o el reviewer está fallando.

### Tasa de rechazo silencioso

De los PRs que mergeas, cuántos tuviste que revertir o corregir después. Esta es la métrica traicionera. Un PR que "pasó" review pero después causa problemas es peor que uno que rechazas de entrada, porque ya está en producción.

### Iteraciones por task

Cuántas vueltas da el loop antes de producir un output aceptable. Si el promedio es 1-2, tu skill y tus criterios están bien calibrados. Si es 4+, algo anda mal: o el skill es ambiguo, o los issues no son realmente autofix-candidates.

### Tasa de abandono

Cuántos tasks el loop no puede resolver y abandona. Algo de abandono es sano (no todo es fixeable por un agente). Pero si abandona más del 50%, tus criterios de "fixeable" están muy laxos.

### Consumo de cuota

Cuánto de tu límite de uso consume cada run. Esto es lo que te dice si el loop es sostenible o si te va a dejar sin cuota a mitad de mes.

### Script para sacar las métricas

Este script parsea tu `AGENTS.md` y genera las métricas:

```bash
#!/bin/bash
# metrics.sh - Parsea el archivo de métricas del loop
# Uso: ./metrics.sh [path/to/loop-metrics.md]
#
# Espera una tabla con este formato:
# | Task | Status | Iterations | Tests Added | Coverage After | Lint | Errors |

FILE="${1:-loop-metrics.md}"

if [ ! -f "$FILE" ]; then
    echo "Error: $FILE no encontrado"
    exit 1
fi

# awk parsea la tabla: columna 3 = Status, 4 = Iterations, 5 = Tests Added
# Ignora headers y separadores (líneas con ---)
read -r done_count abandoned iterations tests_added <<< "$(awk -F'|' '
    /\| done \|/    { done++; iter += $4+0; tests += $5+0 }
    /\| abandoned \|/ { aband++ }
    END { printf "%d %d %d %d", done+0, aband+0, iter+0, tests+0 }
' "$FILE")"

total=$((done_count + abandoned))

echo "=== Métricas del Loop ==="
echo ""
echo "Tasks completadas:  $done_count"
echo "Tasks abandonadas:  $abandoned"
echo "Total tasks:        $total"
echo "Iteraciones totales: $iterations"
echo "Tests agregados:    $tests_added"

echo ""
echo "=== Tasas ==="

if [ "$total" -gt 0 ]; then
    echo "Tasa de completación:          $((done_count * 100 / total))%"
    echo "Tasa de abandono:              $((abandoned * 100 / total))%"
    echo "Iteraciones promedio por task:  $((iterations / total))"
else
    echo "Sin tasks procesadas"
fi

echo ""
echo "=== Para llenar manualmente ==="
echo "PRs mergeados sin cambios: ___"
echo "PRs que necesitaron corrección post-merge: ___"
echo "Tasa de rechazo silencioso: ___"
```

### Cómo usar estos números para iterar

Los números solos no sirven de nada si no los usas para cambiar algo:

- **Tasa de aceptación baja** -> Revisa las instrucciones del maker. Probablemente le falta contexto o los criterios del skill son ambiguos.
- **Tasa de abandono alta** -> Afloja los criterios de "autofix-candidate" en el skill de triage. Estás mandando issues que no son fixeables.
- **Muchas iteraciones por task** -> El maker está dando vueltas. Agrega restricciones más claras: máximo de archivos, máximo de líneas cambiadas, tipos de fix permitidos.
- **Rechazo silencioso alto** -> Tu reviewer no está atrapando los problemas. Endurece sus instrucciones adversariales.
- **Consumo de cuota excesivo** -> Reduce el scope del loop. Menos issues por run, criterios más estrictos, menos iteraciones máximas.

## El sub-agent reviewer: el componente más importante

Separar maker de reviewer es lo que hace que un loop sea confiable.

Un solo agente que escribe y revisa su propio código es como un dev que hace PR y la aprueba él mismo. Técnicamente "funciona". En la práctica, se te pasan cosas.

### Por qué funciona la separación

El maker tiene un incentivo implícito: resolver el issue. Eso lo lleva a tomar atajos. Arreglar el síntoma, no la causa. Hacer un fix que pasa los tests pero introduce deuda técnica. Cambiar un test para que pase en vez de arreglar el código.

El reviewer tiene el incentivo opuesto: encontrar razones para rechazar. Esa tensión es productiva.

### Configuración del reviewer

En `.claude/agents/reviewer.md`:

```markdown
# Reviewer Agent

## Rol
Tu trabajo es encontrar razones para RECHAZAR este cambio. No busques razones para aprobarlo.

## Criterios de rechazo (rechaza si CUALQUIERA aplica)

### Corrección
- El fix resuelve el síntoma pero no la causa raíz
- El fix introduce un nuevo edge case no cubierto por tests
- El fix rompe el contrato de una función pública (cambia signature, return type, o side effects)

### Convenciones
- Viola alguna regla del CLAUDE.md del proyecto
- No sigue el estilo de código del archivo que modifica
- El commit message no sigue conventional commits
- Modifica archivos fuera del scope del issue

### Calidad
- El fix es más complejo de lo necesario (hay una solución más simple)
- Agrega código duplicado que ya existe en el proyecto
- Hardcodea valores que deberían ser configurables
- No maneja errores que el código original sí manejaba

## Qué hacer al rechazar
1. Especifica CUÁL criterio se violó
2. Explica POR QUÉ es un problema (no solo "viola convención X")
3. Sugiere una alternativa concreta
4. Registra el rechazo en AGENTS.md con el motivo

## Qué hacer al aprobar
1. Confirma explícitamente que revisaste TODOS los criterios
2. Lista los archivos que revisaste
3. Registra la aprobación en AGENTS.md
```

### Comparación: instrucciones maker vs reviewer

| Aspecto | Maker | Reviewer |
|---------|-------|----------|
| Objetivo | Resolver el issue | Encontrar razones para rechazar |
| Tono | Constructivo, orientado a solución | Adversarial, escéptico |
| Ante ambigüedad | Toma la decisión más simple | Rechaza y pide clarificación |
| Ante tests que pasan | Continúa al siguiente paso | Verifica que los tests cubren el caso real |
| Límite de scope | "Resuelve con mínimo cambio" | "Verifica que no toca nada fuera de scope" |

### Trade-off: cada sub-agent consume cuota

La pregunta válida: si cada sub-agent consume cuota, vale la pena tener un reviewer separado?

Mi respuesta: depende del costo de un bug en producción.

- **Repo personal, side project**: probablemente no vale la pena. Un solo agente con instrucciones claras alcanza.
- **Repo de trabajo, código que va a producción**: vale la pena cada token. Un bug que se te escapa y llega a prod te cuesta más en tiempo de debugging y revert que lo que costó el reviewer.
- **Loop que abre PRs automáticamente**: de cajón necesitas reviewer. Si el loop abre PRs sin review humano y las mergeas confiando, el reviewer es tu última línea de defensa.

Regla práctica: si mergeaste un PR del loop y tuviste que revertirlo, necesitas reviewer. Si nunca has tenido que revertir, quizás no lo necesitas todavía.

## Condiciones de parada

Un loop sin condiciones de parada claras es un generador de facturas. Estas son las cinco que uso y cómo combinarlas.

### Máximo de iteraciones

El más básico. Hardcoded en el prompt del loop.

```
Máximo 5 iteraciones. Si después de 5 iteraciones no has completado la tarea, 
detente y reporta qué lograste y qué falta.
```

Ni en tu peor día dejes un loop sin esto. Es tu circuit breaker de último recurso.

### Detección de estancamiento

```
Si dos rondas consecutivas no producen cambios nuevos (mismo diff, mismos archivos tocados, mismo resultado de tests), detente inmediatamente. 
Reporta: "Loop estancado después de N iteraciones. Último cambio significativo en iteración M."
```

Esto atrapa el anti-patrón de oscilación: edit, fail, revert, edit, fail, revert. El agente "trabaja" pero no avanza.

### Goal con criterio explícito

```
/goal Los tests en src/handlers/ pasan todos (exit code 0) 
El linter no reporta errores nuevos 
El diff total es menor a 50 líneas
```

El `/goal` le da al agente una definición concreta de "terminé". Sin esto, "sigue hasta que funcione" es una invitación al desastre.

### Token budget en el prompt

```
Usa máximo 10k tokens para esta tarea. Si llegas al 80% del budget (8k tokens) 
sin haber completado, detente y reporta progreso parcial.
```

Esto no es un hard limit (Claude Code no te da un contador de tokens en el prompt), pero el agente lo respeta razonablemente. No es perfecto pero es mejor que nada.

### TTL: scheduled task con expiración

Si usas `CronCreate` o `/loop` con intervalo, agrega un mecanismo de expiración:

```
Este loop expira el 2026-06-17. Después de esa fecha, no ejecutes más runs.
Al expirar, escribe un resumen final en AGENTS.md con métricas acumuladas.
```

### Combinándolas para un loop útil

{{< mermaid >}}
flowchart TD
    A[Inicio de iteración] --> B{¿Máximo iteraciones?}
    B -->|Sí| C[STOP: Límite alcanzado]
    B -->|No| D{¿Estancamiento detectado?}
    D -->|Sí| E[STOP: Loop estancado]
    D -->|No| F[Ejecutar tarea]
    F --> G{¿Goal cumplido?}
    G -->|Sí| H[EXIT: Éxito]
    G -->|No| I{¿Token budget excedido?}
    I -->|Sí| J[STOP: Budget agotado]
    I -->|No| K{¿TTL expirado?}
    K -->|Sí| L[STOP: Tiempo expirado]
    K -->|No| A
{{< /mermaid >}}

En el prompt del loop completo se ve así:

```
## Condiciones de parada (respeta TODAS)
1. Máximo 10 iteraciones por run
2. Si 2 iteraciones consecutivas no producen progreso, detente
3. Goal: todos los tests pasan Y lint clean Y diff < 100 líneas
4. Budget estimado: 15k tokens por run
5. Este loop expira el 2026-06-17

Si CUALQUIERA de estas condiciones se cumple, detente inmediatamente.
Escribe resultado en AGENTS.md antes de terminar.
```

Si una condición falla, otra te atrapa.

## Checklist antes de lanzar un loop

Antes de darle enter a cualquier loop, revisa esto:

- [ ] Condiciones de parada definidas (máximo iteraciones + al menos una más)
- [ ] SKILL.md con criterios concretos, no vibes
- [ ] Worktree configurado si el loop toca código
- [ ] Progress file (AGENTS.md) inicializado
- [ ] Reviewer agent configurado si el loop abre PRs
- [ ] MCP servers necesarios configurados en settings.json
- [ ] Estimación de consumo de cuota por run

Si te falta alguno, no lances el loop. Vas a gastar tokens debuggeando lo que debiste configurar antes.

## Lo que sigue

En el siguiente post de la serie vamos a ver los guardrails: cómo hacer que tus loops se comporten incluso cuando no estás mirando. Skills que actúan como constraints, hooks que validan antes de ejecutar, y patterns para que el agente se detenga cuando debe.

Si quieres el ángulo no-técnico, el argumento de por qué los loops importan para cómo trabajamos, lee [El Loop Que Nadie Testea] en Substack.

