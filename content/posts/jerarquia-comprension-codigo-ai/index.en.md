---
title: "The code comprehension hierarchy: why LSP is just the beginning for AI coding tools"
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
Most AI coding tools understand your code at the level of a junior with grep, when the best ones operate four layers above. Let's talk about the full hierarchy, and why it matters in your day to day.
{{< /lead >}}

When an AI coding tool needs to know where a function is used, most do exactly what you would do with `grep`. Scan text, find matches, and hope the result makes sense. Some make the **jump to LSP** and get real semantic answers, types, definitions, exact references, but almost none operate at the level where they truly understand your system's architecture, the dependencies between services, the transitive impact of a change.

The difference between these layers is not incremental, it's qualitative, and understanding it changes how you evaluate, configure, and trust your tools.

{{< mermaid >}}
graph BT
    A["Plain text<br/>(the file as a string)"] --> B["Pattern matching<br/>(grep, regex, ripgrep)"]
    B --> C["AST / Tree-sitter<br/>(syntactic structure)"]
    C --> D["LSP<br/>(language semantics:<br/>types, definitions, references)"]
    D --> E["Hybrid graph<br/>(embedded type resolution +<br/>persistence + cross-repo)"]
    E --> F["Runtime-enriched graph<br/>(production traces validating<br/>static relationships)"]
    F --> G["Full system comprehension<br/>(structure + runtime + intent +<br/>evolution + operational context)"]

    style A fill:#1a1a2e,stroke:#e94560,color:#eee
    style B fill:#1a1a2e,stroke:#e94560,color:#eee
    style C fill:#16213e,stroke:#0f3460,color:#eee
    style D fill:#0f3460,stroke:#533483,color:#eee
    style E fill:#533483,stroke:#e94560,color:#eee
    style F fill:#e94560,stroke:#fff,color:#fff
    style G fill:#ff6b6b,stroke:#fff,color:#fff
{{< /mermaid >}}

Each layer adds context, but costs more in complexity and resources. The engineering judgment is not "use the highest layer", it's **"use the right layer for the problem"**, and that's what we're going to break down.

## The lower layers: text and pattern matching

This is where every AI coding tool started, and where many still operate... Read files, stuff them into the model's context, and hope it infers relationships. For a single file this works surprisingly well, but for a 200-file project, more files in the context means **more tokens, more noise, and worse response quality**.

The first upgrade is pattern matching: `grep`, `ripgrep`, regular expressions that are fast, universal, and dependency-free, since they remain the right tools for searching string literals, configuration constants, or an exact error message. The problem shows up when you use them for semantic questions:

```bash
# "Where is processPayment used?"
$ rg "processPayment" --type ts

src/api/payments.ts:42:  await processPayment(order)
src/api/payments.ts:15:  // TODO: refactor processPayment to handle retries
src/domain/billing.ts:88: export async function processPayment(order: Order)
src/tests/payments.test.ts:23:  describe("processPayment", () => {
src/utils/logger.ts:67:  logger.info("processPayment completed")
src/types/events.ts:12:  type: "processPayment"
```

**Six results, only two are actual uses of the function** (line 42 and the test), the rest are a comment, the definition itself, a string in a logger, and a type literal. A human filters this in seconds, an AI tool pays tokens for every line and sometimes hallucinates relationships where none exist.

There's an intermediate level worth mentioning: parsing the syntactic structure of code without resolving types or references, **Tree-sitter operates here**, it understands that `processPayment` is a function call and not a comment, but it doesn't know which type it belongs to or who invokes it from another file. **It's a step forward, but not the massive leap that solves everything.**

The real trap is that AI tools use grep for everything because it's the easiest thing to implement, and the model compensates with statistical inference for what it lacks in real comprehension. Sometimes it gets it right, sometimes it hallucinates with confidence, and **you can't tell which without checking by hand**.

## LSP: the layer that changed the game

**LSP is a JSON-RPC protocol where a language server maintains a semantic model of the code and responds to individual queries**, Microsoft created it in 2016 to solve a combinatorial problem: before, supporting M languages across N editors required M×N integrations. **LSP converts that to M+N**, one language server per language that talks to any editor that understands the protocol.

What matters for AI tools is what kind of answers it gives, an LSP request and its response look like this:

**Request:** "where is the symbol at line 42, column 15 defined?"

```json
{
  "method": "textDocument/definition",
  "params": {
    "textDocument": {"uri": "file:///src/api/payments.ts"},
    "position": {"line": 42, "character": 15}
  }
}
```

**Response:** exact location, not text, pure semantics

```json
{
  "uri": "file:///src/domain/billing.ts",
  "range": {
    "start": {"line": 87, "character": 2},
    "end": {"line": 87, "character": 16}
  }
}
```

**LSP doesn't return text, it returns resolved locations**, that difference is what makes it useful for AI tools: instead of guessing where a function lives, the AI gets the exact answer, consumes fewer tokens, and hallucinates less.

### What LSP gives an AI tool

Four capabilities raise the bar:

1. **Go to Definition / Find References**: The AI stops guessing and gets the exact location, fewer tokens wasted on irrelevant files, less hallucination about relationships that don't exist.
2. **Diagnostics**: The AI gets immediate feedback on type errors after every edit, it can self-correct without waiting for you to compile or run tests.
3. **Hover / Type Info**: The AI knows that `user` is `User | null`, it doesn't have to infer it from context with the risk of getting it wrong.
4. **Call Hierarchy**: Who calls whom, to understand the blast radius before a refactor, not perfect, but enormously better than grep.

Up to here, everything any "how to use LSP with your AI tool" tutorial says, and it's all true, LSP is a real and measurable upgrade over grep. The problem is that most of those tutorials stop here, as if LSP were the ceiling, it's not, and understanding why requires looking at its design limitations.

### Where LSP falls short

These aren't complaints or bugs, they're protocol design decisions that make it excellent for editors and limited for AI agents.

- **Stateless by design**: Every request is independent, LSP doesn't remember that 30 seconds ago you asked about the same function. An AI agent that needs to build progressive understanding of a module has to make N individual requests and assemble the puzzle on its own. There's no concept of "explain this module's architecture" because the protocol wasn't designed for that question.

- **Project scope, not system scope**: LSP understands a workspace, but if your system is 15 microservices with shared contracts via gRPC or events, LSP sees each repository as an isolated universe. "Who consumes this endpoint?" is a question LSP can't answer if the consumer lives in another repo. For an editor that's fine, you know which repo you're in, but for an AI agent that should understand the complete system, it's a fundamental blind spot.

- **Syntax and types, not intent**: LSP knows that `retryCount` is a `number` and that it's used in 4 places, but it doesn't know that the payments team bumped it from 3 to 5 last week because there was an incident with the payment provider. Business semantics don't exist at this level, and for many engineering decisions that's exactly the semantics that matter.

- **One-to-one request/response**: An AI agent that wants to understand a complete flow, from the HTTP handler to the database, has to chain 10+ call hierarchy requests, one by one, manually assembling the trace. There's no "give me the full flow of this request" since the protocol wasn't designed for compound queries, and latency accumulates.

- **No memory between sessions**: Every time you open the editor, the language server re-indexes from scratch, there's no accumulated understanding. For a human that's transparent, the language server indexes in seconds and you don't notice, but for an AI agent that operates in short sessions and needs project context from the first prompt, it's context that gets lost and rebuilt every time.

{{< mermaid >}}
graph LR
    subgraph "What LSP sees"
        A[File A] --> B[File B]
        B --> C[File C]
    end

    subgraph "What LSP doesn't see"
        D[Service X<br/>in another repo]
        E[Shared gRPC<br/>contract]
        F[Event<br/>on the bus]
    end

    C -.->|"❌ invisible"| D
    C -.->|"❌ invisible"| E
    B -.->|"❌ invisible"| F

    style D fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
    style E fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
    style F fill:#1a1a2e,stroke:#e94560,color:#eee,stroke-dasharray: 5 5
{{< /mermaid >}}

**LSP gives you an excellent window into one room, but it doesn't give you the building's floor plan.**

## Hybrid graph: where LSP stops being enough

LSP answers questions about your code, a hybrid graph lets you ask questions that LSP doesn't know how to formulate.

With LSP you ask: "who calls `processPayment`?", with a graph you ask: "if I change the `PaymentRequest` contract, which services break, through which call chain, and which of those paths go through a retry?"

That second question requires three things LSP doesn't have by design: **persistent relationships, transitivity, and cross-boundary visibility**.

### What a hybrid graph is and why it's not just "LSP + a database"

The keyword is "hybrid" because this layer doesn't depend on a language server running as an external process. Tools like codebase-memory-mcp take a different approach: they use tree-sitter to parse the syntactic structure of 158 languages and layer on top their own type resolution, implemented in C, inspired by the algorithms of language servers like tsserver, pyright, and gopls, but embedded in a single binary... No external processes, no API keys, no per-project setup.

The result is persisted in a graph database (SQLite with zstd compression) that survives between sessions, can be shared across team members via a repo artifact, and updates incrementally through a file watcher. Nodes are the entities that matter: functions, classes, modules, HTTP routes, Kubernetes resources, Kustomize modules, **and the relationships go beyond what LSP can express**:

- **Control flow:** `CALLS`, `ASYNC_CALLS`
- **Structure:** `IMPORTS`, `IMPLEMENTS`, `INHERITS`, `DEFINES_METHOD`
- **Cross-service:** `HTTP_CALLS` (with confidence scoring), `EMITS`, `LISTENS_ON` (for pub-sub, Socket.IO, EventEmitter)
- **Infrastructure:** `CONFIGURES`, `WRITES` (for Dockerfiles and Kubernetes manifests)
- **Quality:** `SIMILAR_TO` (near-clone detection via MinHash), `FILE_CHANGES_WITH` (change coupling)
- **Cross-repo:** `CROSS_*` edges connecting nodes across multiple indexed repositories

This is what a real graph looks like: 23,827 nodes and 51,526 edges from codebase-memory-mcp's own repo, with functionality clusters visible as galaxies and relationship types filtered in the sidebar.

![Real code graph with 23K nodes and 51K edges showing functionality clusters](codebase-memory-mcp-graph-ui.png)

The difference with LSP is best seen in a side-by-side example.

**LSP:** find-references for processPayment

```json
// textDocument/references
// Result: 7 locations in 3 files within the same project
```

**Hybrid graph:** transitive impact of changing processPayment

```cypher
MATCH (f:Function {name:"processPayment"})<-[:CALLS*1..5]-(caller)
RETURN caller.qualified_name, caller.service, length(path) as depth
-- Result: 23 functions across 8 services, max depth 4
```

Same initial question, but the graph answers **what actually matters**: not just who calls this function directly, but what breaks if you change it, and how many hops away. And it answers in under 1ms, because the relationships are already computed and persisted, not recalculated on every query.

### What it enables for AI tools

Five capabilities that are impossible with LSP alone:

- **Cross-service impact analysis**: Before a refactor, the AI knows exactly what breaks beyond the current project, no guessing, no scanning text in other repos, it traverses the graph and gives you the list with the complete dependency chain. `CROSS_*` edges connect repos that LSP sees as isolated universes.

- **Architectural comprehension**: The AI can answer "how is this system organized?" without reading 200 files. It sees the functionality clusters, the communication patterns between services, the couplings. It can tell you that the payments service has a fan-out of 12 direct dependents and that's a risk, without you having to ask.

- **Structural anomaly detection**: Functions with excessive fan-out, circular dependencies between modules, dead code that nothing invokes, near-clones that should be a shared abstraction. The graph reveals patterns that no amount of grep or LSP queries would reveal because they require a global view.

- **Infrastructure as a first-class citizen**: A Dockerfile and a Kubernetes manifest aren't "text files" to the graph, they're nodes with relationships to the code they configure. If you change the port your Go service listens on, the graph knows there's a Kubernetes `Service` and an `Ingress` that reference that port. LSP doesn't know Kubernetes exists.

- **Persistent context between sessions**: Unlike LSP which re-indexes every time, the graph persists on disk and is shared as a repo artifact. Today's AI can compare against yesterday's graph with `detect_changes`, see which symbols were affected in the latest diff with risk classification. It doesn't rebuild the world from scratch every conversation.

{{< mermaid >}}
graph TD
    subgraph "Payments Service"
        PP[processPayment]
        VP[validatePayment]
        PP --> VP
    end

    subgraph "Orders Service"
        CO[createOrder]
        CO -->|CALLS| PP
    end

    subgraph "Notifications Service"
        SN[sendNotification]
        SN -->|CALLS| CO
    end

    subgraph "Analytics Service"
        TR[trackRevenue]
        TR -->|CALLS| PP
    end

    subgraph "Infra (K8s)"
        SVC[Service :8080]
        ING[Ingress /api/pay]
        SVC -->|CONFIGURES| PP
        ING -->|ROUTES_TO| SVC
    end

    PP -.->|"change here"| CO
    PP -.->|"impacts here"| SN
    PP -.->|"impacts here"| TR
    PP -.->|"impacts here"| SVC

    style PP fill:#e94560,stroke:#fff,color:#fff
    style CO fill:#533483,stroke:#e94560,color:#eee
    style SN fill:#533483,stroke:#e94560,color:#eee
    style TR fill:#533483,stroke:#e94560,color:#eee
    style SVC fill:#0f3460,stroke:#e94560,color:#eee
    style ING fill:#0f3460,stroke:#e94560,color:#eee
{{< /mermaid >}}

A change in `processPayment` directly impacts Orders, Analytics, and the Kubernetes Service that exposes that port and transitively impacts Notifications and the Ingress. LSP tells you about the 7 references within the Payments service, the hybrid graph tells you about the 8 services and 2 infrastructure resources that break if you change the contract.

## Runtime-enriched graph: what the code doesn't tell you

A hybrid graph gives you the complete structure of the system, but structure tells you what *could* happen, not what *does* happen. The next layer closes that gap by feeding the graph with production data.

Tools like codebase-memory-mcp already expose an `ingest_traces` interface designed to accept external traces and validate `HTTP_CALLS` edges that were detected statically. The interface exists, the contract is defined, but the implementation that actually creates edges in the graph from traces hasn't been written yet. If you call it today, it responds with `"Runtime edge creation from traces not yet implemented"` for this particular tool. The pieces to build this layer exist separately: your OpenTelemetry pipeline already has the traces in Tempo, your hybrid graph already has the static edges with confidence scoring, but the bridge connecting them can't be crossed yet.

This, again, is not a criticism, it's an observation about where we are in the maturity of these tools. The fact that the interface is already defined says a lot about where the authors are heading, and when that implementation arrives, an AI tool that integrates traces with the graph will be able to tell you things no previous layer can:

- "This function is called 50,000 times a day but your refactor only has tests for the happy path."
- "This endpoint has a p99 of 3 seconds and the bottleneck is in the third function of the call chain, not the first one where you're looking."
- "The graph says service C depends on B, but in production that path hasn't been executed in 6 months. It's probably dead code."
- "There's an `HTTP_CALLS` edge that static analysis didn't detect but traces show it exists at runtime via a dynamic dispatch."

**Structure gives you the map, traces tell you where people actually walk.** Today we have the map and we have the footsteps, but in different databases that don't talk to each other.

## The final stage: full system comprehension

Above the runtime-enriched graph there's one more layer, and it's the one no tool fully covers today. It's what a real engineer carries in their head after years on a system: they don't just know what the code does and how it behaves in production, they know ***why it exists the way it does***.

Full system comprehension is the convergence of five signals:

1. **Static structure**: the code graph with dependencies, types, and cross-service relationships. Already exists.
2. **Dynamic behavior**: traces, metrics, real traffic patterns. Starting to integrate.
3. **Intent**: why this code exists, what decision motivated it, what ADR backs it, what ticket requested the change. Tools like codebase-memory-mcp's `manage_adr` touch this, but integration with tickets, PRs, and design conversations is just beginning.
4. **Temporal evolution**: how the system changed, what was tried before and failed, which PRs touched this area, who is the real expert on this module (not the nominal owner, but the one who commits). Git history has the data, but no AI tool crosses it well with the structural graph today.
5. **Operational context**: deploy topology, feature flags, incident history, who wakes up at 3am if this fails. The information exists in runbooks, PagerDuty, and the team's collective memory, but it's not connected to the code.

{{< mermaid >}}
graph TD
    subgraph "Today"
        S[Static structure<br/>hybrid graph] 
        R[Runtime<br/>traces + metrics]
    end

    subgraph "Emerging"
        I[Intent<br/>ADRs, tickets, PRs]
        E[Evolution<br/>git history, change coupling]
    end

    subgraph "Future"
        O[Operational context<br/>incidents, deploys, ownership]
    end

    S --> C[Full system<br/>comprehension]
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

No AI tool operates at this full layer today, but the pieces exist: hybrid graphs cover structure, OpenTelemetry covers runtime, ADR tools touch intent, and `FILE_CHANGES_WITH` in the graphs is a first step toward temporal evolution.

### The engineering judgment

The answer is not always the highest layer, and that's what separates someone who understands the hierarchy from someone who reads the tutorial and enables everything:

- **Grep** is still the right answer for searching a constant in a configuration file, an exact error message, or a lost TODO. It's fast, has no dependencies, and for literal text searches nothing beats it.
- **LSP** is perfect for refactors within a project with well-defined types, for navigating a new codebase, for the AI to auto-correct type errors in real time.
- **Hybrid graphs** are what you need when the problem crosses service boundaries, when you need architectural comprehension, or when the question is "what impact does this change have?" and the answer involves more than one repository.
- **Runtime-enriched graphs** are for when the question isn't about code structure but about production behavior, and you need real data to make the right call.
- **Full system comprehension** is for when the question is "should we make this change?", not "can we make this change?", and it involves history, intent, and operational context on top of code.

The AI coding tools that internalize this hierarchy, that know when to use grep and when they need an enriched graph, will separate themselves from the rest. The ones that treat LSP as a silver bullet will produce refactors that compile but break systems.

When you evaluate an AI coding tool, don't ask if it has LSP. Ask what level of the hierarchy it operates at, and whether it knows when to move up or down.
