---
title: "La jerarquía de comprensión de código: por qué LSP es solo el principio para los AI coding tools"
weight: 1
tags: ["lsp", "ai-engineering", "platform-engineering", "devex", "architecture"]
date: "2026-07-15T13:26:00-06:00"
showDateUpdated: false
cascade:
  showDate: false
  showAuthor: false
  invertPagination: true
---

{{< lead >}}
La mayoría de los AI coding tools entienden tu código al nivel de un junior con grep, cuando los mejores operan cuatro capas arriba. Organicé estas capas en siete niveles para entender qué separa a una herramienta de otra.
{{< /lead >}}

Cuando un AI coding tool necesita saber dónde se usa una función, la mayoría hace exactamente lo que harías tú con `grep`. Recorre texto, busca coincidencias, y espera que el resultado tenga sentido. Algunos dan el **salto a LSP** y obtienen respuestas semánticas reales, tipos, definiciones, referencias exactas, pero casi ninguno opera al nivel donde realmente entiende la arquitectura de tu sistema, las dependencias entre servicios, el impacto transitivo de un cambio.

La diferencia entre estas capas no es incremental, es cualitativa, y entenderla cambia cómo evalúas, configuras y confías en tus herramientas.

{{< mermaid >}}
graph BT
    A["Texto plano<br/>(el archivo como string)"] --> B["Pattern matching<br/>(grep, regex, ripgrep)"]
    B --> C["AST / Tree-sitter<br/>(estructura sintáctica)"]
    C --> D["LSP<br/>(semántica de lenguaje:<br/>tipos, definiciones, referencias)"]
    D --> E["Grafo híbrido<br/>(type resolution embebida +<br/>persistencia + cross-repo)"]
    E --> F["Grafo enriquecido con runtime<br/>(traces de producción validando<br/>relaciones estáticas)"]
    F --> G["Comprensión de sistema<br/>(estructura + runtime + intención +<br/>evolución + contexto operacional)"]

    style A fill:#1a1a2e,stroke:#e94560,color:#eee
    style B fill:#1a1a2e,stroke:#e94560,color:#eee
    style C fill:#16213e,stroke:#0f3460,color:#eee
    style D fill:#0f3460,stroke:#533483,color:#eee
    style E fill:#533483,stroke:#e94560,color:#eee
    style F fill:#e94560,stroke:#fff,color:#fff
    style G fill:#ff6b6b,stroke:#fff,color:#fff
{{< /mermaid >}}

Cada capa agrega contexto, pero cuesta más en complejidad y recursos. El juicio de ingeniería no es "usa la capa más alta", es **"usa la capa correcta para el problema"**, y para eso vamos a desarmar cada una.

## Las capas bajas: texto y pattern matching

Es donde empezaron todos los AI coding tools, y donde muchos siguen... Leer archivos, meterlos en el contexto del modelo, y esperar que infiera relaciones. Para un archivo individual funciona sorprendentemente bien, pero para un proyecto de 200 archivos, más archivos en el contexto significa **más tokens, más ruido, y peor calidad de respuesta**.

El primer upgrade es pattern matching: `grep`, `ripgrep`, expresiones regulares que son rápidas, universales y sin dependencias, dado que siguen siendo las herramientas correctas para buscar strings literales, constantes de configuración, o un error message exacto. El problema aparece cuando lo usas para preguntas semánticas:

```bash
# "¿Dónde se usa processPayment?"
$ rg "processPayment" --type ts

src/api/payments.ts:42:  await processPayment(order)
src/api/payments.ts:15:  // TODO: refactor processPayment to handle retries
src/domain/billing.ts:88: export async function processPayment(order: Order)
src/tests/payments.test.ts:23:  describe("processPayment", () => {
src/utils/logger.ts:67:  logger.info("processPayment completed")
src/types/events.ts:12:  type: "processPayment"
```

**Seis resultados, solo dos son usos reales de la función** (línea 42 y el test), el resto son un comentario, la definición misma, un string en un logger, y un type literal. Un humano filtra esto en segundos, un AI tool paga tokens por cada línea y a veces alucina relaciones donde no las hay.

Existe un nivel intermedio que vale mencionar: parsear la estructura sintáctica del código sin resolver tipos ni referencias, **Tree-sitter opera aquí**, entiende que `processPayment` es una llamada a función y no un comentario, pero no sabe a qué tipo pertenece ni quién la invoca desde otro archivo. **Es un paso adelante, pero no es el salto enorme que resuelve todo.**

La trampa real es que los AI tools usan grep para todo porque es lo más fácil de implementar, y el modelo compensa con inferencia estadística lo que le falta en comprensión real. A veces acierta, a veces alucina con confianza, y **tú no puedes distinguir cuándo sin revisar a mano**.

## LSP: la capa que cambió el juego

**LSP es un protocolo JSON-RPC donde un language server mantiene un modelo semántico del código y responde a queries individuales**, Microsoft lo creó en 2016 para resolver un problema combinatorio: antes, soportar M lenguajes en N editores requería M×N integraciones. **LSP lo convierte en M+N**, un language server por lenguaje que habla con cualquier editor que entienda el protocolo.

Lo que importa para AI tools es qué tipo de respuestas da, un request de LSP y su respuesta se ven así:

**Request:** "¿dónde está definido el símbolo en línea 42, columna 15?"

```json
{
  "method": "textDocument/definition",
  "params": {
    "textDocument": {"uri": "file:///src/api/payments.ts"},
    "position": {"line": 42, "character": 15}
  }
}
```

**Response:** ubicación exacta, no texto, semántica pura

```json
{
  "uri": "file:///src/domain/billing.ts",
  "range": {
    "start": {"line": 87, "character": 2},
    "end": {"line": 87, "character": 16}
  }
}
```

**LSP no te devuelve texto, te devuelve ubicaciones resueltas**, esa diferencia es lo que lo hace útil para AI tools: en vez de adivinar dónde vive una función, el AI recibe la respuesta exacta, consume menos tokens, y alucina menos.

### Lo que LSP le da a un AI tool

Cuatro capacidades elevan el nivel:

1. **Go to Definition / Find References**: El AI deja de adivinar y recibe la ubicación exacta, menos tokens desperdiciados en archivos irrelevantes, menos alucinación sobre relaciones que no existen.
2. **Diagnostics**: El AI recibe feedback inmediato de errores de tipo después de cada edit, puede autocorregirse sin esperar a que tú compiles o corras tests.
3. **Hover / Type Info**: El AI sabe que `user` es `User | null`, no tiene que inferirlo del contexto con riesgo de equivocarse.
4. **Call Hierarchy**: Quién llama a quién, para entender el blast radius antes de un refactor, no perfecto, pero enormemente mejor que grep.

Hasta aquí, todo lo que dice cualquier tutorial de "cómo usar LSP con tu AI tool", todo es cierto, LSP es un upgrade real y medible sobre grep. El problema es que la mayoría de esos tutoriales se detienen aquí, como si LSP fuera el techo, no lo es, y entender por qué requiere ver sus limitaciones de diseño.

### Donde LSP se queda corto

Estas no son quejas ni bugs, son decisiones de diseño del protocolo que lo hacen excelente para editores y limitado para AI agents.

- **Stateless por diseño**: Cada request es independiente, LSP no recuerda que hace 30 segundos preguntaste por la misma función. Un AI agent que necesita construir comprensión progresiva de un módulo tiene que hacer N requests individuales y armar el rompecabezas por su cuenta. No hay concepto de "explícame la arquitectura de este módulo" porque el protocolo no fue diseñado para esa pregunta.

- **Scope de proyecto, no de sistema**: LSP entiende un workspace, pero si tu sistema son 15 microservicios con contratos compartidos via gRPC o eventos, LSP ve cada repositorio como un universo aislado. "¿Quién consume este endpoint?" es una pregunta que LSP no puede contestar si el consumidor vive en otro repo. Para un editor eso está bien, tú sabes en qué repo estás, pero para un AI agent que debería entender el sistema completo, es un punto ciego fundamental.

- **Sintaxis y tipos, no intención**: LSP sabe que `retryCount` es un `number` y que se usa en 4 lugares, pero no sabe que el equipo de payments lo subió de 3 a 5 la semana pasada porque hubo un incidente con el proveedor de pagos. La semántica de negocio no existe en este nivel, y para muchas decisiones de ingeniería es exactamente la semántica que importa.

- **Request/response uno-a-uno**: Un AI agent que quiere entender un flujo completo, del HTTP handler hasta la base de datos, tiene que encadenar 10+ requests de call hierarchy, uno por uno, armando el trace manualmente. No hay un "dame el flujo completo de esta request" dado que el protocolo no fue diseñado para queries compuestas, y la latencia se acumula.

- **Sin memoria entre sesiones**: Cada vez que abres el editor, el language server reindexa desde cero, no hay comprensión acumulada. Para un humano eso es transparente, el language server indexa en segundos y tú no lo notas, pero para un AI agent que opera en sesiones cortas y necesita contexto del proyecto desde el primer prompt, es contexto que se pierde y se reconstruye cada vez.

{{< mermaid >}}
graph LR
    subgraph "Lo que LSP ve"
        A[Archivo A] --> B[Archivo B]
        B --> C[Archivo C]
    end

    subgraph "Lo que LSP no ve"
        D[Servicio X<br/>en otro repo]
        E[Contrato gRPC<br/>compartido]
        F[Evento<br/>en el bus]
    end

    C -.->|"❌ invisible"| D
    C -.->|"❌ invisible"| E
    B -.->|"❌ invisible"| F

    style D fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
    style E fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
    style F fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
{{< /mermaid >}}

**LSP te da una ventana excelente hacia una habitación, pero no te da el plano del edificio.**

## Grafo híbrido: donde LSP deja de alcanzar

LSP responde preguntas sobre tu código, un grafo híbrido te permite hacer preguntas que LSP no sabe formular.

Con LSP preguntas: "¿quién llama a `processPayment`?", con un grafo preguntas: "si cambio el contrato de `PaymentRequest`, ¿qué servicios se rompen, a través de qué cadena de llamadas, y cuáles de esas rutas pasan por un retry?"

Esa segunda pregunta requiere tres cosas que LSP no tiene por diseño: **relaciones persistentes, transitividad, y visibilidad cross-boundary**.

### Qué es un grafo híbrido y por qué no es solo "LSP + base de datos"

La palabra clave es "híbrido" porque esta capa no depende de un language server corriendo como proceso externo. Herramientas como codebase-memory-mcp toman un enfoque diferente: usan tree-sitter para parsear la estructura sintáctica de 158 lenguajes y le montan encima una capa propia de resolución de tipos, implementada en C, inspirada en los algoritmos de language servers como tsserver, pyright y gopls, pero embebida en un solo binario... Sin procesos externos, sin API keys, sin setup por proyecto.

El resultado se persiste en una base de datos de grafos (SQLite con compresión zstd) que sobrevive entre sesiones, se comparte entre miembros del equipo via un artefacto en el repo, y se actualiza incrementalmente con un watcher de archivos. Los nodos son las entidades que importan: funciones, clases, módulos, rutas HTTP, recursos de Kubernetes, módulos de Kustomize, **las relaciones van más allá de lo que LSP puede expresar**:

- **Control de flujo:** `CALLS`, `ASYNC_CALLS`
- **Estructura:** `IMPORTS`, `IMPLEMENTS`, `INHERITS`, `DEFINES_METHOD`
- **Cross-service:** `HTTP_CALLS` (con confidence scoring), `EMITS`, `LISTENS_ON` (para pub-sub, Socket.IO, EventEmitter)
- **Infraestructura:** `CONFIGURES`, `WRITES` (para Dockerfiles y manifiestos de Kubernetes)
- **Calidad:** `SIMILAR_TO` (detección de near-clones via MinHash), `FILE_CHANGES_WITH` (change coupling)
- **Cross-repo:** edges `CROSS_*` que conectan nodos entre múltiples repositorios indexados

Así se ve un grafo real: 23,827 nodos y 51,526 edges del propio repo de codebase-memory-mcp, con clusters de funcionalidad visibles como galaxias y los tipos de relación filtrados en el sidebar.

![Grafo de código real con 23K nodos y 51K edges mostrando clusters de funcionalidad](codebase-memory-mcp-graph-ui.png)

La diferencia con LSP se ve mejor en un ejemplo lado a lado.

**LSP:** find-references de processPayment

```json
// textDocument/references
// Resultado: 7 ubicaciones en 3 archivos del mismo proyecto
```

**Grafo híbrido:** impacto transitivo de cambiar processPayment

```cypher
MATCH (f:Function {name:"processPayment"})<-[:CALLS*1..5]-(caller)
RETURN caller.qualified_name, caller.service, length(path) as depth
-- Resultado: 23 funciones en 8 servicios, profundidad máxima 4
```

Misma pregunta inicial, pero el grafo contesta **lo que realmente importa**: no solo quién llama a esta función directamente, sino qué se rompe si la cambias, y a cuántos saltos de distancia. Y lo contesta en menos de 1ms, porque las relaciones ya están computadas y persistidas, no se recalculan en cada query.

### Qué habilita para AI tools

Cinco capacidades que son imposibles con LSP solo:

- **Análisis de impacto cross-service**: Antes de un refactor, el AI sabe exactamente qué se rompe más allá del proyecto actual, no adivina, no escanea texto en otros repos, traversa el grafo y te da la lista con la cadena de dependencia completa. Los edges `CROSS_*` conectan repos que LSP ve como universos aislados.

- **Comprensión arquitectónica**: El AI puede contestar "¿cómo está organizado este sistema?" sin leer 200 archivos. Ve los clusters de funcionalidad, los patrones de comunicación entre servicios, los acoplamientos. Puede decirte que el servicio de pagos tiene un fan-out de 12 dependientes directos y que eso es un riesgo, sin que tú se lo preguntes.

- **Detección de anomalías estructurales**: Funciones con fan-out excesivo, dependencias circulares entre módulos, código muerto que nada invoca, near-clones que deberían ser una abstracción compartida. El grafo revela patrones que ninguna cantidad de grep o LSP queries revelaría porque requieren una vista global.

- **Infraestructura como ciudadano de primera clase**: Un Dockerfile y un manifiesto de Kubernetes no son "archivos de texto" para el grafo, son nodos con relaciones hacia el código que configuran. Si cambias el puerto en el que escucha tu servicio Go, el grafo sabe que hay un `Service` de Kubernetes y un `Ingress` que referencian ese puerto. LSP no sabe que Kubernetes existe.

- **Contexto persistente entre sesiones**: A diferencia de LSP que reindexa cada vez, el grafo persiste en disco y se comparte como artefacto del repo. El AI de hoy puede comparar contra el grafo de ayer con `detect_changes`, ver qué símbolos se afectaron en el último diff con clasificación de riesgo. No reconstruye el mundo desde cero en cada conversación.

{{< mermaid >}}
graph TD
    subgraph "Servicio Payments"
        PP[processPayment]
        VP[validatePayment]
        PP --> VP
    end

    subgraph "Servicio Orders"
        CO[createOrder]
        CO -->|CALLS| PP
    end

    subgraph "Servicio Notifications"
        SN[sendNotification]
        SN -->|CALLS| CO
    end

    subgraph "Servicio Analytics"
        TR[trackRevenue]
        TR -->|CALLS| PP
    end

    subgraph "Infra (K8s)"
        SVC[Service :8080]
        ING[Ingress /api/pay]
        SVC -->|CONFIGURES| PP
        ING -->|ROUTES_TO| SVC
    end

    PP -.->|"cambio aquí"| CO
    PP -.->|"impacta aquí"| SN
    PP -.->|"impacta aquí"| TR
    PP -.->|"impacta aquí"| SVC

    style PP fill:#e94560,stroke:#fff,color:#fff
    style CO fill:#533483,stroke:#e94560,color:#eee
    style SN fill:#533483,stroke:#e94560,color:#eee
    style TR fill:#533483,stroke:#e94560,color:#eee
    style SVC fill:#0f3460,stroke:#e94560,color:#eee
    style ING fill:#0f3460,stroke:#e94560,color:#eee
{{< /mermaid >}}

Un cambio en `processPayment` impacta directamente a Orders, Analytics, y al Service de Kubernetes que expone ese puerto y transitivamente impacta a Notifications y al Ingress. LSP te dice las 7 referencias dentro del servicio de Payments, el grafo híbrido te dice los 8 servicios y los 2 recursos de infraestructura que se rompen si cambias el contrato.

## Grafo enriquecido con runtime: lo que el código no te dice

Un grafo híbrido te da la estructura completa del sistema, pero la estructura te dice lo que *podría* pasar, no lo que *pasa*. La siguiente capa cierra esa brecha alimentando el grafo con datos de producción.

Herramientas como codebase-memory-mcp ya exponen una interfaz `ingest_traces` diseñada para aceptar traces externos y validar edges `HTTP_CALLS` que se detectaron estáticamente. La interfaz existe, el contrato está definido, pero la implementación que realmente crea edges en el grafo a partir de traces aún no está escrita. Si lo llamas hoy, te responde `"Runtime edge creation from traces not yet implemented"` para el caso de esta herramienta. Las piezas para construir esta capa existen por separado: tu pipeline de OpenTelemetry ya tiene los traces en Tempo, tu grafo híbrido ya tiene los edges estáticos con confidence scoring, pero el puente que los conecta todavía no se puede cruzar.

Esto, nuevamente, no es una crítica, es una observación sobre dónde estamos en la madurez de estas herramientas. El hecho de que la interfaz ya esté definida dice mucho sobre hacia dónde van los autores, y cuando esa implementación llegue, un AI tool que integre traces con el grafo va a poder decirte cosas que ninguna capa anterior puede:

- "Esta función se llama 50,000 veces al día pero tu refactor solo tiene tests para el happy path."
- "Este endpoint tiene un p99 de 3 segundos y el cuello de botella está en la tercera función de la call chain, no en la primera donde estás buscando."
- "El grafo dice que el servicio C depende de B, pero en producción esa ruta no se ha ejecutado en 6 meses. Probablemente es código muerto."
- "Hay un edge `HTTP_CALLS` que el análisis estático no detectó pero los traces muestran que existe en runtime via un dispatch dinámico."

**La estructura te dice el mapa, los traces te dicen por dónde camina la gente de verdad.** Hoy tenemos el mapa y tenemos los pasos, pero en bases de datos diferentes que no se hablan.

## La etapa final: comprensión de sistema completa

Arriba del grafo enriquecido con runtime hay una capa más, y es la que ninguna herramienta cubre completamente hoy. Es lo que un verdadero ingeniero lleva en la cabeza después de años en un sistema: no solo sabe qué hace el código y cómo se comporta en producción, sino ***por qué existe así***.

La comprensión de sistema completa es la convergencia de cinco señales:

1. **Estructura estática**: el grafo de código con dependencias, tipos, y relaciones cross-service. Ya existe.
2. **Comportamiento dinámico**: traces, métricas, traffic patterns reales. Empezando a integrarse.
3. **Intención**: por qué existe este código, qué decisión lo motivó, qué ADR lo respalda, qué ticket pidió el cambio. Herramientas como el `manage_adr` de codebase-memory-mcp tocan esto, pero la integración con tickets, PRs y conversaciones de diseño apenas empieza.
4. **Evolución temporal**: cómo cambió el sistema, qué se intentó antes y falló, qué PRs tocaron esta área, quién es el expert real de este módulo (no el owner nominal, sino el que commitea). Git history tiene los datos, pero ningún AI tool los cruza bien con el grafo estructural hoy.
5. **Contexto operacional**: topología de deploy, feature flags, historial de incidentes, quién se despierta a las 3am si esto falla. La información existe en runbooks, PagerDuty, y la memoria colectiva del equipo, pero no está conectada al código.

{{< mermaid >}}
graph TD
    subgraph "Hoy"
        S[Estructura estática<br/>grafo híbrido] 
        R[Runtime<br/>traces + métricas]
    end

    subgraph "Emergiendo"
        I[Intención<br/>ADRs, tickets, PRs]
        E[Evolución<br/>git history, change coupling]
    end

    subgraph "Futuro"
        O[Contexto operacional<br/>incidentes, deploys, ownership]
    end

    S --> C[Comprensión<br/>de sistema<br/>completa]
    R --> C
    I --> C
    E --> C
    O --> C

    style S fill:#533483,stroke:#e94560,color:#eee
    style R fill:#e94560,stroke:#fff,color:#fff
    style I fill:#0f3460,stroke:#533483,color:#eee
    style E fill:#0f3460,stroke:#533483,color:#eee
    style O fill:#1a1a2e,stroke:#0f3460,color:#eee
    style C fill:#ff6b6b,stroke:#fff,color:#fff
{{< /mermaid >}}

Ningún AI tool opera en esta capa completa hoy, pero las piezas existen: grafos híbridos cubren estructura, OpenTelemetry cubre runtime, herramientas de ADR tocan intención, y `FILE_CHANGES_WITH` en los grafos es un primer paso hacia evolución temporal.

### El juicio de ingeniería

La respuesta no es siempre la capa más alta, y eso es lo que separa al que entiende la jerarquía del que lee el tutorial y habilita todo:

- **Grep** sigue siendo la respuesta correcta para buscar una constante en un archivo de configuración, un error message exacto, o un TODO perdido. Es rápido, no tiene dependencias, y para búsquedas de texto literal no hay nada mejor.
- **LSP** es perfecto para refactors dentro de un proyecto con tipos bien definidos, para navegar una codebase nueva, para que el AI autocorrija errores de tipo en tiempo real.
- **Grafos híbridos** son lo que necesitas cuando el problema cruza fronteras de servicio, cuando necesitas comprensión arquitectónica, o cuando la pregunta es "¿qué impacto tiene este cambio?" y la respuesta involucra más de un repositorio.
- **Grafos con runtime** son para cuando la pregunta no es sobre estructura del código sino sobre comportamiento en producción, y necesitas datos reales para tomar la decisión correcta.
- **Comprensión de sistema** es para cuando la pregunta es "¿deberíamos hacer este cambio?", no "¿podemos hacer este cambio?", e involucra historia, intención, y contexto operacional además de código.

Los AI coding tools que internalicen esta jerarquía, que sepan cuándo usar grep y cuándo necesitan un grafo enriquecido, van a separarse del resto. Los que le metan LSP a todo como silver bullet van a producir refactors que compilan pero rompen sistemas.

Cuando evalúes un AI coding tool, no preguntes si tiene LSP. Pregunta en qué nivel de la jerarquía opera, y si sabe cuándo bajar o subir de nivel.
