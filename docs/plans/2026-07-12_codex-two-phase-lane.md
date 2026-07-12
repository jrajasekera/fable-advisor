# Codex as sole implementation lane, split into plan/implement phases

**Date:** 2026-07-12
**Status:** Approved, ready for implementation
**Fork:** `jrajasekera/fable-advisor` (upstream: `DannyMac180/fable-advisor`)

## Problem

Two things are wrong with the plugin as installed:

1. The default implementation lane is `grok-implementer` (Grok 4.5 via the Grok CLI). We have no Grok
   subscription, so the default lane always fails with `STATUS: unavailable` and every task pays a
   wasted round trip before being re-routed.
2. `codex-implementer` runs a single GPT-5.6 Sol invocation at high reasoning for the whole task.
   High-reasoning Sol is the expensive way to do the token-heavy part of implementation — typing out
   the code. The plugin preaches cost discipline for the architect but doesn't apply it one level
   down, inside the lane.

## Solution

Delete the Grok lane. Make codex the sole implementation lane, and split it into two phases that use
different models: **GPT-5.6 Sol at high reasoning plans, GPT-5.6 Terra at medium reasoning implements.**

The expensive reasoning model thinks once and produces a plan; the cheaper model inherits that plan
and does the typing. This mirrors the plugin's own architect/implementer split, applied within the
lane.

## Verified CLI behavior

All of this was tested against the installed `codex-cli 0.144.1` before the design was accepted — it
is not assumed:

- `gpt-5.6-terra` is a valid model slug for this account. (`gpt-5.6-sol` already is; it is the
  `~/.codex/config.toml` default.)
- `codex exec resume <thread_id> --model <other-model>` **carries conversation context across a model
  swap.** Test: Sol invented a codeword in a `read-only` planning turn; Terra resumed that thread,
  recalled the codeword — which existed nowhere but Sol's turn — and wrote it to disk. Terra can
  therefore inherit Sol's plan rather than being re-briefed from scratch.
- The `thread_id` is emitted in the `thread.started` event of `codex exec --json` output. This is the
  only way to obtain it.
- **`codex exec resume` accepts a narrower flag set than `codex exec`.** It has *no* `--sandbox` and
  *no* `--cd`. The sandbox must be set with `-c sandbox_mode=workspace-write`, and the working
  directory is inherited from the shell, so the agent must `cd` first. The obvious spelling
  (`--sandbox workspace-write`) fails with a usage error.

## Design

### 1. Fork and repoint

Work lives in `jrajasekera/fable-advisor`, cloned at `~/plugins/fable-advisor`. Claude Code's
marketplace is repointed at the fork so plugin updates pull from us, not upstream. `upstream` remains
a remote so Dan's later improvements can be cherry-picked.

Version bumps to **4.0.0** — removing a lane is a breaking change.

### 2. Delete the Grok lane

- Delete `agents/grok-implementer.md`.
- `skills/orchestration/SKILL.md`: the lane table collapses to two rows — **Implementation** (codex,
  two-phase) and **Judgment** (fable-advisor).
- Strip Grok from `plugin.json`, `.claude-plugin/marketplace.json`, and `README.md`.

**Consequence to be explicit about:** the doctrine's "race both lanes on the same spec and pick the
stronger diff" pattern dies with the Grok lane — only one implementation family remains. Replace that
guidance with: for high-stakes work, either keep it with the architect or consult `fable-advisor`
before committing. Genuine cross-vendor review survives regardless, because a Claude architect is
still reviewing OpenAI's diff.

### 3. Two-phase `codex-implementer`

**Phase 1 — plan. Sol, high reasoning, `read-only` sandbox.**

Sol receives the five-part spec and produces a concrete implementation plan. The `read-only` sandbox
is the enforcement mechanism, not a suggestion: Sol *physically cannot* write code, so the planning
phase cannot quietly become an implementation phase. Sol can still read any file in the tree.

Run with `--json` to capture `thread_id` from `thread.started`, and `-o` to capture the plan text.

**The gate.** Sol ends its plan with an explicit verdict token:

- `SPEC-OK` → the agent flows straight into phase 2. No round trip in the normal case.
- `SPEC-PROBLEM` → the agent **stops** and returns `STATUS: spec-gap` with the plan and the
  objection. A bad spec is caught before Terra builds the wrong thing.

Fix decisions stay with the caller. This matches the plugin's existing rule that architectural
problems belong upstream, not inside the lane.

**Phase 2 — implement. Terra, medium reasoning, `workspace-write` sandbox.**

`codex exec resume <thread_id> --model gpt-5.6-terra` inherits Sol's plan and writes the code.

**Fallback if `thread_id` extraction fails:** phase 2 runs as a fresh Terra invocation with Sol's plan
text pasted into the prompt — still two-phase, just re-briefed rather than resumed. The report must
say so. This must never silently degrade to a single-phase run.

**Preserved from the current agent:** the no-silent-fallback rule. If `codex` is missing, unauthenticated,
or a model is inaccessible to the account, the agent returns `STATUS: unavailable` with the exact error
and never substitutes itself. A cross-vendor lane that quietly becomes a Claude lane is worse than a
loud failure.

Each phase gets its own 600s timeout, using the existing portable-`timeout` idiom (macOS has no
`timeout` without coreutils).

**Independent verification is unchanged:** the agent reads the diff and re-runs the spec's
verification command itself. Codex's claim of success is not evidence.

### 4. Report format

Extends the existing `CODEX REPORT` block with two lines:

```
CODEX REPORT
STATUS: complete | partial | spec-gap | timeout | unavailable
MODELS: sol/high (plan) -> terra/medium (implement)   # or "terra re-briefed (resume failed)"
OBJECTIVE: [one line]
PLAN: [one-line summary of Sol's plan]
CHANGES: [file — one-line summary, per file, from the actual diff]
VERIFIED: [verification command re-run — actual output evidence]
CODEX SAID: [one-line summary of the final message; note disagreement with the diff]
GAPS: [spec ambiguities, unfinished items, or "none"]
```

`STATUS: spec-gap` is new — it is how the plan gate reports a refusal to proceed.
`MODELS:` exists so any substitution (resume failure, re-brief) is always visible to the caller.

## Non-goals

- Not touching `agents/fable-advisor.md`. The advisor's model and behavior are unchanged.
- Not adding a Claude implementation lane. Codex is the sole implementer.
- Not making the planning phase user-approvable per task. The gate fires only on real problems.
