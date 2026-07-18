# Data & AI Architecture Decisions

A public record of real architecture decisions from enterprise data and AI platform work — Google Cloud, distributed analytics, multi-agent systems, and data governance. Each entry is a genuine decision from production systems I've designed and led, not a hypothetical.

> **What this is:** ADRs and case studies, sanitized of anything company-identifying, kept honest about trade-offs and rejected alternatives. See [`docs/sanitization-guidelines.md`](docs/sanitization-guidelines.md) for exactly what was removed and why.
>
> **What this isn't:** No proprietary source code, no internal names, URLs, credentials, or production thresholds.

## Repository Structure

```
data-ai-architecture-decisions/
├── adr/                  Architecture Decision Records, numbered sequentially across all projects
├── case-studies/         Full case study per project — context, stack, diagrams, decision index
├── diagrams/             C4-model diagram sources (.mmd) referenced by case studies
├── templates/            Reusable ADR template
└── docs/                 Process docs (sanitization rules, etc.)
```

## Case Studies

| Case Study | Summary |
| :--- | :--- |
| [AI Observability Platform](case-studies/ai-observability-platform.md) | A closed-loop Observe → Optimize → Change system unifying query telemetry, live cluster metadata, and infrastructure signals, with deterministic detection and LLM-assisted synthesis |

*(Additional case studies — data quality platform, conversational AI copilot — will land here as they're written up.)*

## Architecture Decision Records

| # | Decision | Project |
| :--- | :--- | :--- |
| [0001](adr/0001-deterministic-detection-over-llm-qualification.md) | Deterministic qualification and prioritization before LLM analysis | AI Observability Platform |
| [0002](adr/0002-cloud-run-service-and-job-over-gke.md) | Cloud Run Service and Job over GKE | AI Observability Platform |
| [0003](adr/0003-mcp-tool-contracts-over-in-process-tools.md) | MCP tool contracts over in-process tools | AI Observability Platform |
| [0004](adr/0004-git-native-knowledge-over-vector-rag.md) | Git-native knowledge over vector RAG | AI Observability Platform |

## Architectural Principles Behind These Decisions

1. **Managed services where they remove operational ownership** — not by default, but when the workload doesn't justify the control they'd cost.
2. **Deterministic controls on any path that autonomously files a ticket or otherwise creates engineering work.** Generative AI adds value in reasoning and synthesis, not in gating.
3. **Explicit trust boundaries** between read access, diagnostics, and anything that writes a change.
4. **Reviewable knowledge over opaque retrieval** wherever exact-match correctness matters more than fuzzy recall.

## About Me

Kuldeep Kumar — Data & AI Platform Architect, 16+ years across enterprise data warehousing, cloud data platforms, and AI/agentic systems on Google Cloud. [LinkedIn](https://www.linkedin.com/in/kuldeepk-10211190)

## License

Documentation in this repository is shared under the [MIT License](LICENSE). No proprietary source code is included. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how new ADRs and case studies get added.
