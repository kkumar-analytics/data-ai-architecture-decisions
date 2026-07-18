# Case Study: AI Observability Platform (Observe → Optimize → Change)

**Related decisions:** [ADR-0001](../adr/0001-deterministic-detection-over-llm-qualification.md) · [ADR-0002](../adr/0002-cloud-run-service-and-job-over-gke.md) · [ADR-0003](../adr/0003-mcp-tool-contracts-over-in-process-tools.md) · [ADR-0004](../adr/0004-git-native-knowledge-over-vector-rag.md)
**Diagram sources:** [`diagrams/ai-observability-context.mmd`](../diagrams/ai-observability-context.mmd) · [`diagrams/ai-observability-container.mmd`](../diagrams/ai-observability-container.mmd)

## 1. Problem and Business Context

A distributed SQL analytics platform serving enterprise-scale reporting workloads fails in expensive-to-diagnose ways: query latency spikes, memory pressure, resource-pool saturation, and plan regressions after upgrades. Before this platform existed:

- Query and plan telemetry lived in a search/telemetry index.
- Live cluster truth (session state, resource pools, plan cache) lived only in the database engine's own metadata views.
- Corroborating host and infrastructure signals lived in a separate dashboarding tool.
- Root-cause knowledge lived in engineers' heads, with no durable, reviewable record.
- The handoff from "we found something" to "there's a tracked change ticket" was manual and inconsistent.

The mandate was to collapse **Observe → Optimize → Change** into one governed loop: cut mean-time-to-detect and mean-time-to-resolve, and turn every confirmed finding into both a durable change ticket and a reusable operator knowledge asset.

## 2. System Context

```mermaid
C4Context
title System Context — AI Observability Platform

Person(engineer, "Platform Engineer", "Investigates data-platform incidents interactively")

System(platform, "AI Observability Platform", "Closed-loop Observe -> Optimize -> Change system")

System_Ext(telemetry, "Telemetry Index", "Query and plan-cache metrics at query-fingerprint grain")
System_Ext(dbengine, "Distributed SQL Engine", "Live cluster metadata: sessions, resource pools, plan cache")
System_Ext(dashboards, "Metrics Dashboards", "Host and infrastructure signal corroboration")
System_Ext(ticketing, "Ticketing System", "Change-management workflow")
System_Ext(llm, "LLM Provider", "Generative reasoning for optimization rationale and ticket synthesis")

Rel(engineer, platform, "Investigates via chat UI")
Rel(platform, telemetry, "Reads query and plan telemetry")
Rel(platform, dbengine, "Reads live metadata", "read-only")
Rel(platform, dashboards, "Reads host and infra signals")
Rel(platform, ticketing, "Files structured change tickets")
Rel(platform, llm, "Requests optimization rationale and ticket synthesis")
```

## 3. Container View

```mermaid
C4Container
title Container Diagram — AI Observability Platform

Person(engineer, "Platform Engineer")

System_Boundary(platform, "AI Observability Platform") {
  Container(ui, "Interactive Investigation UI", "Cloud Run Service", "Chat-based entry point for human-driven investigation")
  Container(router, "Specialist Agent Router", "Multi-agent framework", "Routes intents to Observe / Optimize / Change specialists")
  Container(job, "Scheduled Detection Job", "Cloud Run Job", "Hourly deterministic sweep and severity scorer")
  Container(obsAgent, "Observability Agent", "LLM agent", "Investigates telemetry, reads knowledge cards")
  Container(optAgent, "Optimization Agent", "LLM agent", "Produces optimization rationale for qualified candidates")
  Container(chgAgent, "Change Agent", "LLM agent", "Generates structured change tickets")
  Container(mcp, "Tool-Contract Layer", "MCP servers", "Shared, scoped access to telemetry / metadata / dashboards / ticketing")
  ContainerDb(lock, "Distributed Lock and Dedup Store", "Document store", "Prevents overlapping runs and duplicate tickets")
  ContainerDb(kb, "Knowledge Cards", "Git-versioned Markdown", "Reviewable RCA and concept knowledge")
}

System_Ext(telemetry, "Telemetry Index")
System_Ext(dbengine, "Distributed SQL Engine Metadata")
System_Ext(dashboards, "Metrics Dashboards")
System_Ext(ticketing, "Ticketing System")
System_Ext(scheduler, "Scheduler")

Rel(engineer, ui, "Uses")
Rel(ui, router, "Routes request")
Rel(scheduler, job, "Triggers hourly")
Rel(job, mcp, "Reads telemetry and metadata via")
Rel(job, lock, "Acquires lock, checks dedup key")
Rel(router, obsAgent, "OBSERVE")
Rel(router, optAgent, "OPTIMIZE")
Rel(router, chgAgent, "CHANGE")
Rel(obsAgent, mcp, "Calls")
Rel(optAgent, mcp, "Calls")
Rel(chgAgent, mcp, "Calls")
Rel(obsAgent, kb, "Reads")
Rel(chgAgent, lock, "Writes dedup lock")
Rel(mcp, telemetry, "Queries")
Rel(mcp, dbengine, "Queries", "read-only")
Rel(mcp, dashboards, "Queries")
Rel(mcp, ticketing, "Creates tickets")
```

## 4. Component Stack

| Architectural Layer | Technology Selected | Engineering Reason |
| :--- | :--- | :--- |
| **Ingestion / Transport** | Aggregated search/telemetry index at query-fingerprint grain; metrics-dashboard panel queries; chat-based entry point; scheduler-triggered batch job | Hot-path telemetry already existed at the right grain; the dashboard layer supplies host/pool corroboration the telemetry index can't; human and scheduled triggers reuse one tool surface instead of duplicating detection logic |
| **Orchestration / Logic** | Multi-agent framework (ReAct-style specialists) + intent router; MCP tool-contract layer; deterministic qualify-and-prioritize scorer | Narrow per-agent tool scopes avoid "god-agent" failure modes; the shared tool layer gives IDE/CLI tooling parity with the hosted UI; the LLM is deliberately excluded from the alerting critical path — see [ADR-0001](../adr/0001-deterministic-detection-over-llm-qualification.md) |
| **Storage / Indexing** | Search/telemetry index; read-only database-engine metadata views; a document store for distributed locks and ticket dedup; git-versioned Markdown knowledge cards | Separates hot telemetry, live cluster truth, durable change state, and operator knowledge — each gets the consistency model it actually needs |
| **Observability / Compute** | Python runtime; optional tracing instrumentation; CI publishing two container images from one shared codebase (interactive service, scheduled monitor); per-caller query hard-limits | Empirically tuned budgets stop runaway agent loops; dry-run-by-default and schema validation in CI keep both deploys and the knowledge base honest |

## 5. Deployment Topology

Interactive investigation runs as an HTTP service (chat-style UI) that scales to zero when idle, not an always-on process. The hourly detection sweep runs as a **scheduled batch job** invoked by a cron-style scheduler. Both are built from one shared codebase but packaged as two separate container images, so an image-only update to either surface preserves secrets, networking, and service-account bindings out of band. A document-store-backed distributed lock (tuned to slightly exceed the job's expected runtime) and a content-hash dedup key prevent overlapping runs and duplicate tickets.

A managed multi-agent hosting product was evaluated for both flows and rejected — the production-critical path is a deterministic batch scorer plus a small set of custom tool servers, not an agent-mesh-native workload. A Kubernetes-based deployment was also evaluated and rejected: cluster lifecycle, node pools, and autoscaler tax are unjustified for a workload that is bursty on the batch side and modest in concurrency on the interactive side. Full reasoning: [ADR-0002](../adr/0002-cloud-run-service-and-job-over-gke.md).

## 6. Results in Practice

The same codebase runs both the interactive UI and the unattended hourly monitor. The monitor deterministically qualifies outliers, then hands off to the LLM only for optimization write-ups and structured ticket filing. Operator knowledge is captured as git-versioned Markdown cards — known issues, platform concepts, metadata-view references, dashboard references — with a deterministic index regenerated and schema-validated in CI on every change. There is no vector index and no runtime knowledge-serving database; the generated index is a build artifact of the Markdown, not a compiled catalog agents query separately. Dry-run defaults make the platform safe to run locally with zero production credentials.

## 7. Architectural Principles Behind This System

1. **Managed services where they remove operational ownership** — not by default, but when the workload doesn't justify the control they'd cost.
2. **Deterministic controls on any path that pages a human or files a ticket.** Generative AI adds value in reasoning and synthesis, not in gating.
3. **Explicit trust boundaries** between telemetry access, read-only diagnostics, and anything that writes a change.
4. **Reviewable knowledge over opaque retrieval** wherever exact-match correctness matters more than fuzzy recall.

---
*Pattern summary: multi-agent specialists on a shared MCP tool-contract fabric; git-native operator knowledge; deterministic detection with LLM-assisted synthesis; a dual-runtime deployment aligned to interactive vs. batch control loops.*
