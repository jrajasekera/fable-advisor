---
name: orchestration
description: Routing doctrine for the architect-as-orchestrator pattern — how a session running the smartest model delegates implementation to a cheaper cross-vendor lane to minimize cost. USE WHEN delegating implementation work, routing to the codex-implementer lane, writing a spec for a subagent, deciding whether to consult fable-advisor, managing session cost or token spend, or running any multi-task build where the session is the architect.
---

# Orchestration — the architect's routing doctrine

The session is the architect: it owns requirements, architecture, decomposition, specs, routing, and verification. It should almost never type implementation code. Every implementation task gets routed to the cheapest lane that is adequate for it — escalation is deliberate, per task, never a fixed binding.

## Cost discipline — the prime directive

The session model is the most expensive lane in the system, on both input and output tokens. The whole economic case for this pattern is keeping its token volume low: spend Fable on judgment, spend the codex lane on volume. Three rules follow.

**Emit judgment, not volume.** The architect's output is decomposition, specs, routing decisions, verdicts on diffs, and short reports. It does not type implementation code, test bodies, boilerplate, or config files. A code block longer than an interface signature or a few illustrative lines is a spec that hasn't been delegated yet — stop and delegate it. Fixing a lane's bug by hand is the same failure in disguise: send a corrected spec back to the cheap lane instead.

**Keep the context lean.** Everything in the architect's context is re-read at architect prices on every turn. Delegate broad exploration, codebase searches, and log-grepping to a cheap read-only agent and keep only the conclusions; read files yourself only when the decision genuinely depends on the exact code. Don't paste long files, full diffs, or verbose command output into the conversation when a path reference or an excerpt will do.

**Reason once, then hand off.** Do the hard thinking — the architecture, the interface design, the debugging hypothesis — in one pass, capture it in the spec, and let the cheap lane carry it from there. Re-deriving decisions across turns burns the premium twice.

What stays with the architect regardless of cost: decomposition, interface design, hypothesis selection when debugging, spec writing, lane routing, and judging verification evidence. Those tokens are what the premium is for — everything else is a candidate for delegation.

## The lanes

| Lane | Producer | Invoke | Route here when |
|---|---|---|---|
| Implementation | GPT-5.6 Sol (plan) → GPT-5.6 Terra (implement) | `codex-implementer` agent | All well-specified implementation work. **The only implementation lane.** Requires the [Codex CLI](https://github.com/openai/codex). |
| Judgment | Fable 5 | `fable-advisor` agent | Not an implementation lane. See "Commitment boundaries" below. |

The implementation lane is itself two-phase: Sol at high reasoning reads the code and produces a plan, then Terra at medium reasoning inherits that plan and writes the code. The architect's cost discipline applied one level down — reason once with the expensive model, let the cheap one carry the volume.

Deciding rule: how much does the outcome depend on judgment the spec can't capture? Little → delegate; you will verify anyway. A lot, and mistakes are costly → keep it with the architect, or settle the decision with `fable-advisor` first and *then* delegate the settled spec.

The lane is a different model family than the architect, so every diff gets genuine cross-vendor review for free: OpenAI produces, Claude judges.

The lane's planning phase is also a spec check. If it returns `STATUS: spec-gap`, Sol judged the spec defective and refused to build against it — that is a signal to fix the spec (or consult `fable-advisor`), never to re-send the same spec with firmer wording.

If the lane returns `unavailable` or `timeout`, say so explicitly in your report — never quietly absorb a substitution. If the codex CLI is unavailable entirely, implement with a Claude subagent and state the downgrade plainly.

## The spec contract

Implementers share none of your conversation context. Every delegation prompt carries all five parts:

1. **Objective** — what to build or change, one paragraph
2. **Files** — exact paths to create or modify
3. **Interfaces** — signatures, types, or API shapes the code must match
4. **Constraints** — project conventions, things not to touch
5. **Verification** — the command(s) that prove it works

A spec you can't finish writing is a signal the decision isn't made yet — that's architect work, not a reason to hand the ambiguity to a cheaper model.

## Parallelism

Independent specs (no shared files, no ordering dependency) launch as parallel `codex-implementer` agents in a single message. Sequential chains and single-file surgery stay serial. Each lane invocation uses `mktemp` paths, so parallel runs do not collide.

For high-stakes work there is no second lane to race against — buy confidence with `fable-advisor` on the decision *before* delegating, and with your own diff review after.

## Commitment boundaries

Consult `fable-advisor` (read-only, verdict in under 300 words) at the moments that decide whether the next hour is wasted:

- Before committing to an architecture, data migration, API shape, or refactor strategy
- Whenever the same problem has resisted two distinct attempts
- Once before declaring a multi-step deliverable done

Pass it the decision, the constraints, and the options considered. Act on the verdict or surface the disagreement — never silently ignore it. (If the session itself already runs on Fable, the advisor still earns its keep as a context-clean skeptic reading the actual code.)

## Verification

Reports are claims, not evidence. Before accepting any lane's work: read the diff, and re-run the verification command (or spot-check its quoted output against the working tree). "Should work", "tests should pass", or a report with no command output means the task is not done. A lane that reports a spec gap gets a corrected spec, not a "use your judgment".
