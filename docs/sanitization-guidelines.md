# Sanitization Guidelines

Rules applied when converting internal architecture work into public case studies and ADRs for this repository. Apply these consistently before publishing any new project.

## What Gets Removed or Generalized

| Category | Rule | Example |
| :--- | :--- | :--- |
| **Company / product names** | Never named. Replace with a role-based description of the domain. | ~~"impact.com's copilot"~~ → "an authenticated customer-facing copilot for an ad-tech/analytics platform" |
| **Internal repo, service, or job names** | Replace with the generic role the thing plays. | ~~`copilotproxy`~~ → "the Cloud Run BFF" |
| **URLs / endpoints** | Never included, even internal ones that "look" harmless. | ~~`https://internal.corp/api/...`~~ → omitted entirely |
| **Index / table / dataset names** | Replace with what the index is *for*, not its literal name. | ~~`custom-data_platform-*`~~ → "a dedicated append-only telemetry index" |
| **Credentials, tokens, project IDs, account numbers** | Never included. Not even redacted-but-shaped placeholders that hint at format. | — |
| **Ticket prefixes / issue-tracker project keys** | Replace with a generic reference. | ~~`GENA-4821`~~ → "a tracked change ticket" |
| **Exact production thresholds / tuning constants** | Describe the *shape* of the constraint, not the number, unless the number itself is the architectural point being made. | ~~"a 59-minute lock"~~ → "a bounded lock tuned to slightly exceed the job's expected runtime" |
| **Team / org names, headcount tied to a specific employer** | Generalize to a role description ("a five-person platform team") only if team size itself is load-bearing for the decision (e.g., justifying "no dedicated ops staff"). Otherwise omit. |

## What Is Safe to Keep

These are **not** sensitive and should stay specific — vague-washing them makes the writing weaker without protecting anything:

- **Technology and vendor names** actually used (GCP, Cloud Run, Firestore, Elasticsearch, SingleStore, Grafana, Jira, Dialogflow CX, Vertex AI, LangChain, MCP, etc.). These are public products; naming them is a factual, useful signal to an interviewer.
- **Architectural patterns and terminology** (event sourcing, CQRS, C4 model, Kimball dimensional modeling, idempotent pipelines, watermarking).
- **Publicly known certifications, job titles, and dates** on the resume side.
- **The reasoning itself** — alternatives considered, trade-offs, and consequences should be as concrete and opinionated as the real decision was. Sanitization is about removing *identifying* detail, not softening the argument.

## Process Checklist Before Publishing a New Case Study or ADR

1. Search the draft for the employer's name and any product name unique to that employer — remove or genericize every instance.
2. Search for anything that looks like a URL, internal hostname, or index/table pattern (`*_prod`, `custom-*`, service names with underscores) — genericize.
3. Search for ticket-style tokens (`[A-Z]+-[0-9]+`) — remove or genericize.
4. Search for numbers that look like tuned production constants (durations, thresholds, counts) — ask "does the exact number matter to the argument, or just its existence?" If just its existence, generalize it.
5. Re-read the "why" sections (alternatives, trade-offs) and confirm nothing was diluted in the process — the goal is anonymized, not vague.
6. Have someone unfamiliar with the original system read it and confirm they can't identify the employer from context clues (client names, industry specifics, unusual product combinations).

## Numbering Convention

ADR numbers are sequential **across the whole repository**, not restarted per project. When starting a new case study, check the highest existing ADR number in `adr/` and continue from there.
