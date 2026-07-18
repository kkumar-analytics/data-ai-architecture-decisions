# ADR-0003: MCP Tool Contracts Over In-Process Tools

**Status:** Accepted
**Documentation maturity:** Initial
**Date:** 2026
**Project:** AI Observability Platform
**Deciders:** Platform Architecture Lead

## Context

The platform needs to expose the same capabilities — telemetry queries, read-only database metadata lookups, dashboard queries, ticket creation — to two very different consumers: a hosted multi-agent runtime serving the interactive investigation UI, and engineers working directly in MCP-aware IDEs and CLIs who want to run the same investigations locally. Building these as private, in-process callables inside the hosted application would mean writing the integration twice, and the two implementations would drift the moment either one changed.

There's also a naming problem: several systems expose conceptually similar operations (for example, a counter-reset operation exists for more than one backend). A single agent loaded with every tool from every system risks name collisions and, worse, gives any one agent access to capabilities it has no business using.

## Constraints

- Two consumer types (hosted agent runtime, local developer tooling) must get identical behavior from identical tool contracts — no "close enough" reimplementation.
- Specialist agents should only ever see the tools relevant to their role (observe / optimize / change), not a shared kitchen-sink toolbox.
- Guardrails — dry-run defaults, per-caller query budgets — need to live at a layer both consumer types pass through, not duplicated in each client.

## Decision

Implement every capability as a standalone tool server following the **Model Context Protocol (MCP)**. The hosted agent runtime opens only the servers a given specialist needs; the same servers register in the repository's MCP configuration so IDE and CLI clients get identical tool access without a second implementation.

## Alternatives Considered

1. **In-process tools only, bound directly inside the hosted application.** Fastest path to ship the interactive UI. Rejected: zero reuse for local/IDE investigations, and the two integration paths would inevitably drift from each other over time.
2. **A bespoke HTTP microservice per backend system.** Cleaner for non-AI consumers and a familiar pattern for a traditional services team. Rejected: heavier auth and operational surface per system, and a weaker fit for agent tool-calling schemas than MCP's native tool-contract shape.

## Trade-offs

**Pros of the chosen route:**
- One implementation, many clients — the hosted runtime and local developer tools are guaranteed to behave identically.
- Server-per-system scoping forces explicit boundaries and prevents tool-name collisions across backends that expose similar operations.
- Dry-run behavior and query-budget guardrails live naturally at the tool-server edge, not scattered across callers.

**Cons / risks of the chosen route:**
- Multi-server orchestration adds process and session management overhead compared to a single in-process toolbox.
- Requires ongoing naming and scoping discipline as more tool servers are added.
- Debugging a tool call now spans a process boundary instead of a single stack trace.

## Operational Consequences

Every agent declares an explicit list of tool servers it's allowed to open — access is scoped by design, not by convention. Guardrail resets (dry-run toggles, budget resets) are server-scoped operations. Local onboarding registers the exact same tool servers that production uses, which makes tool-access parity between environments an enforced invariant rather than a hope.
