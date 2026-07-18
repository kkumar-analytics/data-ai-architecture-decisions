# Contributing / Authoring Guide

This is a personal architecture-decisions portfolio, not an open-source project soliciting external contributions. This guide exists so the repo stays consistent as new case studies get added over time — by me, or reviewed by anyone I ask to sanity-check a draft.

## Adding a New Case Study

1. Pick a real, already-implemented architectural decision — not a hypothetical or a "best practice" writeup. See the [ADR Philosophy](docs/sanitization-guidelines.md) for why this matters: fabricated examples don't hold up under interview follow-up questions.
2. Write the case study in `case-studies/<project-name>.md` following the structure of the existing entry: problem/business context, system context diagram, container diagram, component stack table, deployment topology, results, and a links-out decision index.
3. Extract each genuinely separate decision into its own ADR in `adr/`, using [`templates/adr-template.md`](templates/adr-template.md).
4. Add diagram sources to `diagrams/` as `.mmd` files, and embed the same Mermaid code in the case study's fenced code block so it renders on GitHub.
5. Run every new file through the [sanitization checklist](docs/sanitization-guidelines.md#process-checklist-before-publishing-a-new-case-study-or-adr) before committing.
6. Number new ADRs sequentially from the highest existing number in `adr/` — numbering is repo-wide, not per case study.
7. Update the root `README.md` tables (Case Studies and ADRs) to include the new entries.

## Writing Standard for ADRs

- **State the decision, don't narrate the deliberation.** The Decision section is one sentence, stated as a position.
- **Alternatives need real reasons for losing**, not strawmen. If an alternative was genuinely close, say so — that's more credible than "and obviously we chose X."
- **Trade-offs must include real cons.** An ADR with no cons for the chosen route reads as marketing, not architecture.
- **Operational Consequences should name what becomes someone's job** as a direct result — not restate the decision in different words.

## Diagram Standard

- Use the [C4 model](https://c4model.com/) levels — Context and Container are the default; add a Deployment diagram where infrastructure topology is itself part of the story, and Component-level diagrams only where the internal structure of one container is itself the point of an ADR.
- Diagrams are written as styled Mermaid flowcharts (`flowchart`), not raw `C4Context`/`C4Container` syntax — the C4 boxes render too flat and text-heavy on GitHub. Use `classDef` to color-code actor / entry-point / agent / tool / storage / external-system node categories consistently across a case study's diagram set, and keep a small palette (5–6 classes) rather than one-off colors per node.
- Case studies that center on a closed-loop or pipeline process (e.g. detect → diagnose → act) should also include a narrative flow diagram showing that loop end to end — this is usually the single most recruiter-legible diagram in the case study, so it earns its own diagram file.
- Every diagram lives in `diagrams/` as its own `.mmd` file and gets embedded verbatim (same Mermaid source, no drift) in the case study's fenced code block.
- Keep node/participant names in diagrams role-based (`Scheduled_Detection_Job`), matching the sanitization rules — never an internal service name.

## Review Pass Before Publishing

Before pushing a new case study or ADR set, re-read [`docs/sanitization-guidelines.md`](docs/sanitization-guidelines.md) in full and run its checklist. This is the single most important step — everything else here is about consistency, that step is about not leaking anything.
