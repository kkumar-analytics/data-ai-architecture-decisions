# ADR-0004: Git-Native Knowledge Over Vector RAG

**Status:** Accepted
**Documentation maturity:** Initial
**Date:** 2026
**Project:** AI Observability Platform
**Deciders:** Platform Architecture Lead

## Context

An earlier iteration of this platform had a knowledge-serving layer: document loaders, a compiled cache of embeddings, and a dedicated service to query it. In practice, that layer added regeneration drift (the compiled index and the source docs could silently disagree), opaque retrieval (no clean way to know why a given card surfaced), and a second source of truth for facts engineers already trusted more when they were just files in the repo. What operators actually needed was root-cause and concept knowledge that's reviewable in a pull request, greppable by exact key (a known issue ID, a metadata view name, a dashboard name), and durable across model or embedding-model changes.

## Constraints

- RCA knowledge must survive changes to whichever LLM or embedding model the platform uses next — it can't be tied to a specific vector representation.
- Exact-key lookups (a specific known issue, a specific metadata view, a specific dashboard) matter more here than fuzzy semantic recall.
- Whoever curates this knowledge should review it the same way they review code — as a diff, not as a silent index rebuild.

## Decision

Treat the Markdown file itself as the knowledge base: schema-validated front matter plus prose, organized by category (known issues, platform concepts, metadata-view references, dashboard references), with deterministic generated Markdown indexes for discoverability and a small set of "capture" workflows as the growth loop. No vector index. No compiled cache. No runtime knowledge database or dedicated knowledge-base service.

## Alternatives Considered

1. **Vector RAG over incident documents.** Strong for fuzzy recall across loosely related incidents. Rejected as the primary mechanism: weak for the exact-key lookups (a specific query hash, table, or workspace) that RCA work actually depends on, and effectively unreviewable in a code-review workflow.
2. **A dedicated knowledge-base service with compiled caches.** Gives a centralized query API. Rejected: introduces cache-regeneration drift, a second generation pipeline, and a serving layer that duplicates what plain filesystem access already does well for any agent with repo access.

## Trade-offs

**Pros of the chosen route:**
- Git is the review plane — knowledge changes go through the same pull-request scrutiny as code.
- The source implementation defines schema validation and deterministic index-drift checks; those CI workflow files are not included in this public architecture repository.
- Scheduled scrapes can refresh generated reference stubs (e.g., metadata-view definitions) without overwriting hand-authored operator judgment.

**Cons / risks of the chosen route:**
- Any hosted runtime without filesystem access to the repository cannot read these cards — this is an explicit, accepted limitation until that changes.
- Retrieval quality now depends on index discipline and on agents actually following the "read the index, then open cards" protocol — not on embedding similarity doing the work automatically.

## Operational Consequences

Investigations must read the generated index before opening individual cards — this is a protocol requirement, not a suggestion. Curating knowledge (capturing a new finding, a new concept, a new reference view) becomes explicit, named work with its own workflow, not a side effect of usage. The knowledge base grows as reviewed pull requests, which means its quality is bounded by review discipline rather than by retrieval-tuning effort.
