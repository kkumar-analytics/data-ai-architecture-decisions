# ADR-0002: Cloud Run Service and Job Over GKE

**Status:** Accepted
**Date:** 2025
**Project:** AI Observability Platform
**Deciders:** Platform Architecture Lead

## Context

The observability platform needs two distinct execution modes from the same codebase: an **interactive mode**, where an engineer opens a chat-style UI mid-incident and asks a specialist agent to investigate, and an **unattended mode**, where the same detection logic runs on an hourly schedule with no human present. Both modes need the same tool access (telemetry index, database metadata, dashboards, ticketing) and the same agent/scoring code — they differ in trigger and in how long they're allowed to run.

The team operating this system is five engineers, three of whom are GenAI specialists also carrying other platform responsibilities. There is no dedicated infrastructure or SRE function for this specific system.

## Constraints

- No standing Kubernetes cluster already exists for this workload, and no one on the team is staffed to operate one.
- The scheduled detection job is bursty — it runs for a few minutes every hour, not continuously.
- Interactive concurrency is modest — this is an internal investigation tool, not a customer-facing service at scale.
- Local development must work with zero production credentials (a dry-run mode is a hard requirement, not a nice-to-have).
- Cost should scale with actual invocations, not with idle capacity.

## Decision

Run two independently deployable **Cloud Run** artifacts from one container image: a **Cloud Run Service** for interactive, chat-driven investigation, and a **Cloud Run Job** triggered hourly by a managed scheduler for deterministic detection. No Kubernetes cluster, no managed multi-agent hosting product.

## Alternatives Considered

1. **Kubernetes (Deployments + CronJobs).** Genuinely the right call if this system needed long-lived sidecars, custom networking, or fine-grained multi-container co-location. Rejected because cluster lifecycle, node-pool management, and autoscaler tuning are a real ongoing cost that a bursty hourly job and modest interactive load don't justify.
2. **A managed multi-agent hosting product** (the kind of platform that manages agent session lifecycle for you). Attractive on paper for the interactive investigation flow. Rejected because the production-critical path here is a deterministic batch scorer plus a handful of custom tool servers — not an agent-mesh-native workload. Adopting a managed agent runtime would have added a second orchestration paradigm for a system that mostly needs a plain, auditable batch job.
3. **One always-on service handling both interactive and scheduled work.** Simplest to deploy. Rejected because coupling a latency-sensitive interactive surface to a long-running background sweep increases blast radius — a stuck sweep shouldn't be able to degrade the tool an engineer is actively using mid-incident, and vice versa.

## Trade-offs

**Pros of the chosen route:**
- Zero cluster operations — nothing to patch, upgrade, or capacity-plan at the infrastructure layer.
- Clean separation of failure domains: an interactive 5xx and a failed scheduled sweep are different incidents with different blast radii.
- Image-only deploys (new container, same service/job config) keep secrets, networking, and service-account bindings stable across releases.
- Pay-per-invocation cost model fits a genuinely bursty workload.

**Cons / risks of the chosen route:**
- Cold starts on invocation, which matter more for the interactive surface than the batch one.
- Cloud Run Job timeout ceilings put a hard cap on worst-case sweep duration — the detection logic has to fit inside that window.
- Less process-topology control than a cluster offers — no sidecars, no custom scheduling policies, no direct pod-to-pod networking.

## Operational Consequences

On-call reasoning splits cleanly into two domains: interactive-service health and scheduled-job health, each with its own failure signatures. New capability generally means a new Cloud Run revision from the same shared image, not a new cluster manifest or Helm chart. Changing detection cadence is a scheduler configuration change, not a re-architecture. The team explicitly accepted "less infrastructure control" as the price of "no infrastructure team required."
