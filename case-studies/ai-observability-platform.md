# Case Study: AI Observability Platform (Observe → Optimize → Change)

**Related decisions:** [ADR-0001](../adr/0001-deterministic-detection-over-llm-qualification.md) · [ADR-0002](../adr/0002-cloud-run-service-and-job-over-gke.md) · [ADR-0003](../adr/0003-mcp-tool-contracts-over-in-process-tools.md) · [ADR-0004](../adr/0004-git-native-knowledge-over-vector-rag.md)
**Diagram sources:** [`diagrams/ai-observability-context.mmd`](../diagrams/ai-observability-context.mmd) · [`diagrams/ai-observability-loop.mmd`](../diagrams/ai-observability-loop.mmd) · [`diagrams/ai-observability-container.mmd`](../diagrams/ai-observability-container.mmd) · [`diagrams/ai-observability-deployment.mmd`](../diagrams/ai-observability-deployment.mmd)

## 1. Problem and Business Context

A distributed SQL analytics platform serving enterprise-scale reporting workloads fails in expensive-to-diagnose ways: query latency spikes, memory pressure, resource-pool saturation, and plan regressions after upgrades. Before this platform existed:

- Query and plan telemetry lived in Elasticsearch.
- Live cluster truth (session state, resource pools, plan cache) lived only in SingleStore's own metadata views.
- Corroborating host and infrastructure signals lived in Grafana.
- Root-cause knowledge lived in engineers' heads, with no durable, reviewable record.
- The handoff from "we found something" to "there's a tracked change ticket" was manual and inconsistent.

The mandate was to collapse **Observe → Optimize → Change** into one governed loop: cut mean-time-to-detect and mean-time-to-resolve, and turn every confirmed finding into both a durable change ticket and a reusable operator knowledge asset.

## 2. System Context

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px"}}}%%
flowchart TB
    classDef actor fill:#1f2937,stroke:#94a3b8,color:#f8fafc,stroke-width:1.5px
    classDef platform fill:#0f766e,stroke:#5eead4,color:#f0fdfa,stroke-width:3px,font-weight:bold
    classDef external fill:#1e293b,stroke:#64748b,color:#e2e8f0,stroke-width:1.5px

    Engineer(["Platform Engineer"]):::actor

    Platform["AI Observability Platform<br/><small>Closed-loop Observe → Optimize → Change system</small>"]:::platform

    Telemetry[("Elasticsearch<br/><small>query & plan-cache metrics</small>")]:::external
    DBEngine[("SingleStore<br/><small>live cluster metadata</small>")]:::external
    Dashboards[("Grafana<br/><small>host & infra signals</small>")]:::external
    Ticketing[["Jira<br/><small>change management</small>"]]:::external
    LLM{{"Vertex AI · Gemini<br/><small>generative reasoning</small>"}}:::external

    Engineer -->|"investigates via chat UI"| Platform
    Platform -->|"reads query & plan telemetry"| Telemetry
    Platform -->|"reads live metadata · read-only"| DBEngine
    Platform -->|"reads host / infra signals"| Dashboards
    Platform -->|"files structured change tickets"| Ticketing
    Platform -->|"requests optimization rationale"| LLM
```

## 3. The Observe → Optimize → Change Loop

The core product idea in one picture: detection is 100% deterministic, the LLM is only invited in once a candidate has already qualified, and anything below threshold never reaches a human or a ticket queue.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "15px"}}}%%
flowchart LR
    classDef observe fill:#0e7490,stroke:#67e8f9,color:#ecfeff,stroke-width:2px
    classDef optimize fill:#0f766e,stroke:#5eead4,color:#f0fdfa,stroke-width:2px
    classDef change fill:#7c3aed,stroke:#c4b5fd,color:#f5f3ff,stroke-width:2px
    classDef gate fill:#b45309,stroke:#fbbf24,color:#fffbeb,stroke-width:2px,stroke-dasharray: 4 3
    classDef store fill:#334155,stroke:#94a3b8,color:#f1f5f9,stroke-width:1.5px
    classDef discard fill:#1e293b,stroke:#475569,color:#94a3b8,stroke-dasharray: 3 3

    Telemetry[("Elasticsearch<br/><small>query & plan telemetry</small>")]:::store

    subgraph Observe["1 · OBSERVE"]
        Sweep["Deterministic sweep<br/><small>9 core metrics · incidents & patterns</small>"]:::observe
    end

    Gate{"Deterministic<br/>Qualify & Prioritize Scorer"}:::gate

    subgraph Optimize["2 · OPTIMIZE"]
        Explain["Read-only plan diagnosis<br/><small>LLM-assisted rationale</small>"]:::optimize
    end

    subgraph Change["3 · CHANGE"]
        Ticket["Structured change ticket<br/><small>LLM-synthesized · deduped</small>"]:::change
    end

    Discard(["below threshold<br/>no ticket · no LLM call"]):::discard
    KB[("Knowledge Cards<br/><small>git-versioned</small>")]:::store

    Telemetry --> Sweep --> Gate
    Gate -->|"qualified candidate"| Explain
    Gate -.-> Discard
    Explain --> Ticket
    Explain -.->|"reads / informs"| KB
    Ticket -.->|"captures RCA"| KB
```

## 4. Container View

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    classDef actor fill:#1f2937,stroke:#94a3b8,color:#f8fafc,stroke-width:1.5px
    classDef entry fill:#0e7490,stroke:#67e8f9,color:#ecfeff,stroke-width:2px
    classDef agent fill:#0f766e,stroke:#5eead4,color:#f0fdfa,stroke-width:1.5px
    classDef tool fill:#7c3aed,stroke:#c4b5fd,color:#f5f3ff,stroke-width:2px
    classDef store fill:#334155,stroke:#94a3b8,color:#f1f5f9,stroke-width:1.5px
    classDef external fill:#1e293b,stroke:#64748b,color:#e2e8f0,stroke-width:1.5px

    Engineer(["Platform Engineer"]):::actor
    Scheduler(("Cloud Scheduler")):::external

    subgraph Platform["AI Observability Platform"]
        direction TB

        UI["Interactive Investigation UI<br/><small>Cloud Run Service</small>"]:::entry
        Job["Scheduled Detection Job<br/><small>Cloud Run Job · hourly sweep</small>"]:::entry
        Router{"Specialist Agent Router<br/><small>LangChain</small>"}:::entry

        ObsAgent["Observability Agent<br/><small>LLM agent</small>"]:::agent
        OptAgent["Optimization Agent<br/><small>LLM agent</small>"]:::agent
        ChgAgent["Change Agent<br/><small>LLM agent</small>"]:::agent

        MCP["MCP Tool Servers<br/><small>FastMCP</small>"]:::tool

        Lock[("Firestore<br/><small>locks & ticket dedup</small>")]:::store
        KB[("Knowledge Cards<br/><small>git-versioned Markdown</small>")]:::store
    end

    Telemetry[("Elasticsearch")]:::external
    DBEngine[("SingleStore<br/><small>metadata, read-only</small>")]:::external
    Dashboards[("Grafana")]:::external
    Ticketing[["Jira"]]:::external

    Engineer --> UI
    Scheduler -->|"triggers hourly"| Job

    UI --> Router
    Router -->|"OBSERVE"| ObsAgent
    Router -->|"OPTIMIZE"| OptAgent
    Router -->|"CHANGE"| ChgAgent

    Job -->|"sweep"| MCP
    Job -->|"lock"| Lock

    ObsAgent --> MCP
    OptAgent --> MCP
    ChgAgent --> MCP
    ObsAgent -.->|"reads"| KB
    ChgAgent -.->|"dedup"| Lock

    MCP --> Telemetry
    MCP -->|"read-only"| DBEngine
    MCP --> Dashboards
    MCP -->|"creates tickets"| Ticketing
```

## 5. Component Stack

| Architectural Layer | Technology Selected | Engineering Reason |
| :--- | :--- | :--- |
| **Ingestion / Transport** | Elasticsearch at query-fingerprint grain; Grafana panel queries; chat-based entry point (Cloud Run Service); Cloud Scheduler-triggered batch job (Cloud Run Job) | Hot-path telemetry already existed at the right grain; Grafana supplies host/pool corroboration Elasticsearch can't; human and scheduled triggers reuse one MCP tool surface instead of duplicating detection logic |
| **Orchestration / Logic** | LangChain multi-agent framework (ReAct-style specialists) + intent router; MCP tool servers (FastMCP) via `langchain-mcp-adapters`; deterministic qualify-and-prioritize scorer | Narrow per-agent tool scopes avoid "god-agent" failure modes; the shared MCP layer gives IDE/CLI tooling parity with the hosted UI; the LLM (Vertex AI / Gemini) is deliberately excluded from the alerting critical path — see [ADR-0001](../adr/0001-deterministic-detection-over-llm-qualification.md) |
| **Storage / Indexing** | Elasticsearch; read-only SingleStore metadata views; Firestore for distributed locks and ticket dedup; git-versioned Markdown knowledge cards | Separates hot telemetry, live cluster truth, durable change state, and operator knowledge — each gets the consistency model it actually needs |
| **Observability / Compute** | Python runtime; Cloud Build CI/CD (in progress — local dev currently publishes both container images directly to Artifact Registry); Secret Manager for credentials; per-caller query hard-limits | Empirically tuned budgets stop runaway agent loops; dry-run-by-default and schema validation keep both deploys and the knowledge base honest even before CI/CD lands |

## 6. Deployment Topology

Interactive investigation runs as a **Cloud Run Service** (chat-style UI) that scales to zero when idle, not an always-on process. The hourly detection sweep runs as a **Cloud Run Job** invoked by **Cloud Scheduler**. Both are built from one shared codebase but packaged as two separate container images, so an image-only update to either surface preserves secrets, networking, and service-account bindings out of band. A **Firestore**-backed distributed lock (tuned to slightly exceed the job's expected runtime) and a content-hash dedup key prevent overlapping runs and duplicate tickets. CI/CD via **Cloud Build** is on the roadmap; today, both images are built and pushed straight from local development, which is a deliberate near-term trade-off, not an oversight — see the diagram below.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    classDef dev fill:#1e293b,stroke:#64748b,color:#e2e8f0,stroke-width:1.5px
    classDef ci fill:#374151,stroke:#9ca3af,color:#f9fafb,stroke-width:1.5px
    classDef planned fill:#1e293b,stroke:#475569,color:#94a3b8,stroke-width:1.5px,stroke-dasharray: 4 3
    classDef svc fill:#0e7490,stroke:#67e8f9,color:#ecfeff,stroke-width:2px
    classDef job fill:#b45309,stroke:#fbbf24,color:#fffbeb,stroke-width:2px
    classDef shared fill:#334155,stroke:#94a3b8,color:#f1f5f9,stroke-width:1.5px
    classDef actor fill:#1f2937,stroke:#94a3b8,color:#f8fafc,stroke-width:1.5px

    Dev["Local Development<br/><small>shared codebase · dry-run by default</small>"]:::dev
    CloudBuild["Cloud Build<br/><small>CI/CD · planned</small>"]:::planned

    subgraph Registry["Artifact Registry"]
        direction LR
        ImgSvc["service image"]:::ci
        ImgJob["job image"]:::ci
    end
    Dev -->|"push"| ImgSvc
    Dev -->|"push"| ImgJob
    CloudBuild -.->|"future push"| ImgSvc
    CloudBuild -.->|"future push"| ImgJob

    Engineer(["Platform Engineer"]):::actor
    Sched(("Cloud Scheduler<br/><small>hourly</small>")):::actor

    ImgSvc --> Service["Cloud Run Service<br/><small>interactive investigation UI<br/>scales to zero when idle</small>"]:::svc
    ImgJob --> Job["Cloud Run Job<br/><small>scheduled detection sweep<br/>bounded runtime</small>"]:::job

    Engineer -->|"HTTPS"| Service
    Sched -->|"triggers"| Job

    subgraph Shared["Shared Services"]
        direction LR
        LLM{{"Vertex AI · Gemini"}}:::shared
        Secrets[/"Secret Manager"/]:::shared
        Docs[("Firestore<br/><small>locks · tickets · context</small>")]:::shared
    end

    Service --> LLM
    Job --> LLM
    Service --> Secrets
    Job --> Secrets
    Job --> Docs
    Service -.-> Docs
```

A managed multi-agent hosting product was evaluated for both flows and rejected — the production-critical path is a deterministic batch scorer plus a small set of custom tool servers, not an agent-mesh-native workload. A Kubernetes-based deployment was also evaluated and rejected: cluster lifecycle, node pools, and autoscaler tax are unjustified for a workload that is bursty on the batch side and modest in concurrency on the interactive side. Full reasoning: [ADR-0002](../adr/0002-cloud-run-service-and-job-over-gke.md).

## 7. Results in Practice

The same codebase runs both the interactive UI and the unattended hourly monitor, replacing what was previously a fully manual loop: an engineer scanning Grafana/Elasticsearch by hand, copying SQL into SingleStore for a manual `EXPLAIN`, then writing up a Jira ticket from scratch with no dedup and no standard template. Now the monitor deterministically qualifies outliers every hour at zero LLM cost for sweeps that don't clear the bar, and hands off to the LLM only for optimization write-ups and structured ticket filing on the candidates that do.

Operator knowledge is captured as git-versioned Markdown cards — known issues, SingleStore concept references, metadata-view references, Grafana dashboard references — validated against a JSON Schema and compiled to a generated index by a GitHub Actions gate on every pull request that touches the knowledge base, so a card that drifts from the committed index fails CI rather than silently going stale. There is no vector index and no runtime knowledge-serving database; the generated index is a build artifact of the Markdown, not a compiled catalog agents query separately — see [ADR-0004](../adr/0004-git-native-knowledge-over-vector-rag.md). Dry-run defaults make the platform safe to run locally with zero production credentials.

## 8. Architectural Principles Behind This System

1. **Managed services where they remove operational ownership** — not by default, but when the workload doesn't justify the control they'd cost.
2. **Deterministic controls on any path that pages a human or files a ticket.** Generative AI adds value in reasoning and synthesis, not in gating.
3. **Explicit trust boundaries** between telemetry access, read-only diagnostics, and anything that writes a change.
4. **Reviewable knowledge over opaque retrieval** wherever exact-match correctness matters more than fuzzy recall.

---
*Pattern summary: multi-agent specialists on a shared MCP tool-contract fabric; git-native operator knowledge; deterministic detection with LLM-assisted synthesis; a dual-runtime deployment aligned to interactive vs. batch control loops.*
