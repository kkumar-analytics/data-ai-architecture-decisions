# ADR-0001: Deterministic Detection Over LLM Qualification

**Status:** Accepted
**Date:** 2025
**Project:** AI Observability Platform
**Deciders:** Platform Architecture Lead

## Context

The hourly detection sweep decides, for every candidate query or workspace, two things: is this bad enough to act on, and if several things are bad at once, which one goes to an engineer first. Get either answer wrong in the wrong direction and the cost is immediate — either a real incident goes unpaged, or the team drowns in low-value tickets from one noisy workspace and stops trusting the system. This is also the one part of the platform that runs completely unattended: nobody reviews the sweep's output before it decides whether to file a ticket.

Generative models were an obvious candidate for this step — narrative triage ("does this look bad?") is exactly the kind of judgment call an LLM is good at describing. That's precisely why it was rejected for this specific step.

## Constraints

- The qualification step runs hourly, unattended, with no human in the loop before a ticket can be filed.
- A single degraded workspace can generate dozens of candidate anomalies in one sweep; whatever qualifies them must also enforce a sane ticket budget.
- Whatever pages an engineer must be explainable after the fact in one sentence — "it exceeded threshold X on metric Y" — not "the model judged it was likely important."
- Cost and latency at this cadence must be predictable; this step cannot have variable per-call spend.
- The system must not systematically miss the case where *everything* looks bad at once — a purely relative or peer-comparison signal treats uniform degradation as normal.

## Decision

The scheduled job scores every candidate with a **deterministic, reproducible formula**: absolute per-metric floor thresholds combined with a severity score built from magnitude, frequency, breadth, and known plan-warning signals. Selection is two-stage — best candidate per workspace, then a global severity cap across workspaces — enforced with a distributed lock and a content-hash dedup key so overlapping runs can't double-file. The LLM is not part of this decision. It is invoked only *after* a candidate has already qualified, to write the optimization rationale and the structured ticket body.

## Why the LLM Was Excluded From the Critical Detection Path

- **Determinism is a requirement, not a preference, for anything that pages someone.** A threshold-and-formula score gives the same answer on the same input every time, which means a false page can be traced to a specific number, not to a model's mood on a given run. That traceability is what makes on-call trust the system instead of routing around it.
- **Uniform degradation is a known blind spot for narrative or peer-relative judgment.** If every workspace in a cluster degrades together, a comparison against peers — human or model — can conclude "this looks normal for right now." Absolute floor thresholds don't have that failure mode; they don't need a healthy baseline to compare against.
- **Cost and latency at this cadence have to be flat.** An LLM call per candidate, at hourly cadence across every workspace, turns a fixed-cost batch job into a variable-cost one, with no floor on how bad a bad sweep hour gets.
- **Auditability beats sophistication here.** A scoring formula can be unit-tested, back-tested against last quarter's incidents, and retuned by adjusting a number. A model's qualification judgment can only be spot-checked after the fact.
- **This is a values statement about where language reasoning actually adds value.** An LLM is genuinely good at explaining *why* something is a problem and drafting *what to do about it* — and genuinely the wrong tool for deciding *whether* to page someone at 3 a.m.

## Where the LLM Is Used Instead

Once a candidate has qualified through the deterministic scorer, the LLM is invoked for exactly two things: producing the optimization rationale (why this query/plan is degraded, what to try) and generating the structured ticket body in the format the change-management system expects. Both are review-friendly outputs a human reads before acting — not gating decisions a human never sees.

## Alternatives Considered

1. **LLM-as-detector directly on raw telemetry.** Flexible, narrative triage that can reason across loosely related signals. Rejected: unstable qualification thresholds that shift with prompt or model changes, false negatives that are hard to audit after the fact, and unacceptable cost/latency variance for an hourly, SLO-sensitive job.
2. **Pure cross-sectional anomaly / z-score filtering with no absolute floor.** Statistically clean and requires no hand-tuned thresholds. Rejected as the sole mechanism: blind to the uniform-degradation case where every candidate looks "normal" relative to its peers, which is exactly the scenario where paging matters most.

## Trade-offs

**Pros of the chosen route:**
- Every page is auditable to a specific score and threshold, not a model judgment call.
- Stable, predictable paging behavior and flat compute cost at scale.
- Workspace fairness is enforced structurally (per-workspace next-in-line, then a global cap), not left to model discretion.
- LLM spend is concentrated exactly where language reasoning pays off — rationale and ticket prose — not spent on a yes/no gate.

**Cons / risks of the chosen route:**
- Floor thresholds require empirical retuning as workloads evolve; they are not self-adjusting.
- Absolute gates can miss genuinely novel failure modes that don't yet have a historical magnitude signature — until the knowledge base and detection rules catch up.

## Operational Consequences

Improving detection reliability is a data- and threshold-engineering problem, owned like any other tunable production parameter — not a prompt-tuning problem. The optimize and change stages remain LLM-assisted, but strictly downstream of a qualification gate the LLM never touches. Overlapping runs and duplicate tickets are handled as an infrastructure concern (distributed lock plus content-hash dedup), not left to agent behavior or prompt discipline.
