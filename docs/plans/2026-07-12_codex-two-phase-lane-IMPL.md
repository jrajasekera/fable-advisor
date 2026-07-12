# Codex Two-Phase Lane — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make codex the sole implementation lane and split it into a Sol/high planning phase and a Terra/medium implementation phase, with a gate that stops on a bad spec.

**Architecture:** The plugin is entirely markdown — agent prompts plus one skill. There is no code and no test framework, so "tests" here are (a) structural checks on the files (`jq`, `grep`, frontmatter) and (b) one live end-to-end run of the real `codex` CLI against a throwaway git repo. A change is not done until the lane has actually run.

**Tech Stack:** Markdown agent definitions, JSON plugin manifests, `codex-cli` 0.144.1, Claude Code plugin marketplace.

**Spec:** `docs/plans/2026-07-12_codex-two-phase-lane.md`

**Repo:** `~/source/fable-advisor`, branch `codex-two-phase-lane`. `origin` = `jrajasekera/fable-advisor` (SSH), `upstream` = `DannyMac180/fable-advisor`.

## Global Constraints

- **Model slugs are exact:** `gpt-5.6-sol` (planning), `gpt-5.6-terra` (implementation). Verified valid on this account.
- **Reasoning efforts are exact:** `model_reasoning_effort=high` for Sol, `model_reasoning_effort=medium` for Terra.
- **`codex exec resume` has no `--sandbox` and no `--cd` flag.** Sandbox goes through `-c sandbox_mode=workspace-write`; the working directory is inherited from the shell, so `cd` first. Using `--sandbox` on `resume` fails with a usage error.
- **`thread_id` is only obtainable from `codex exec --json`,** in the `thread.started` event.
- **Planning phase is `--sandbox read-only`.** This is the enforcement mechanism that keeps planning from becoming implementation. Never relax it.
- **No silent fallback, ever.** Missing CLI, failed auth, or inaccessible model → `STATUS: unavailable` with the exact error. The agent never implements the task itself.
- **Version is `4.0.0`** in both `plugin.json` and any README references.
- Every file that currently names Grok must stop naming it, except the git history.

---

### Task 1: Rewrite `codex-implementer` as a two-phase agent

**Files:**
- Modify: `agents/codex-implementer.md` (full rewrite of body; frontmatter `name`/`model`/`tools` unchanged)

**Interfaces:**
- Produces: an agent whose report block later tasks and the orchestration doctrine reference — statuses `complete | partial | spec-gap | timeout | unavailable`, and the fields `MODELS:` and `PLAN:` which are new in 4.0.0.

- [ ] **Step 1: Write the file**

Replace the entire contents of `agents/codex-implementer.md` with:

````markdown
---
name: codex-implementer
description: The implementation lane. Runs a two-phase codex flow — GPT-5.6 Sol (high reasoning) plans, GPT-5.6 Terra (medium reasoning) implements — via the OpenAI Codex CLI. Route all well-specified implementation work here: the expensive reasoning model thinks once, the cheaper model does the typing, and both come from a different model family than the Claude architect that reviews the diff. Receives the five-part spec; returns a structured report with verification evidence. Requires the `codex` CLI installed and authenticated — reports a structured error if it is missing, never silently substitutes itself.
model: sonnet
tools: Bash, Read, Grep, Glob
---

# Codex Implementer

You are the implementation lane. You do not write the code yourself — **codex writes it**, in two phases with two different models:

| Phase | Model | Reasoning | Sandbox | Job |
|---|---|---|---|---|
| 1. Plan | `gpt-5.6-sol` | high | `read-only` | Read the code, produce an implementation plan, judge whether the spec is sound |
| 2. Implement | `gpt-5.6-terra` | medium | `workspace-write` | Inherit that plan and write the code |

The split is the point: high-reasoning Sol is the expensive way to type out code, so it thinks once and hands the plan to a cheaper model that does the volume. Terra **resumes Sol's codex thread**, so it inherits the plan as conversation context rather than being re-briefed.

Your job is to deliver the spec faithfully, run both phases, enforce the plan gate, verify the result independently, and report. You exist because a second model family catches what a single vendor's models jointly miss.

## Preflight — no silent fallback

First action, always:

```bash
command -v codex && codex --version
```

If codex is not installed or not authenticated, **stop immediately** and return:

```
CODEX REPORT
STATUS: unavailable
REASON: [codex not found on PATH | auth error — exact message]
```

If either invocation reports that `gpt-5.6-sol` or `gpt-5.6-terra` is unavailable to the current account or workspace, return the same report with `STATUS: unavailable` and preserve the exact access error in `REASON`.

You never implement the task yourself as a fallback. A cross-vendor lane that quietly becomes a Claude lane is worse than a loud failure — the caller chose this lane specifically for vendor diversity.

## The contract

The prompt you receive should contain the five-part spec: **objective, files, interfaces, constraints, verification command**. If parts are missing, do not paper over the gap — the planning phase exists to catch exactly this, so pass it through and let Sol judge it.

## Phase 1 — plan (Sol, high, read-only)

Work from the repository root. Write the spec to a unique prompt file — never inline shell quoting, never a fixed path (parallel lanes on fixed paths corrupt each other):

```bash
SPEC=$(mktemp -t codex-spec.XXXXXX)
PLAN=$(mktemp -t codex-plan.XXXXXX)
EVENTS=$(mktemp -t codex-events.XXXXXX)
FINAL=$(mktemp -t codex-final.XXXXXX)

cat > "$SPEC" << 'SPEC_EOF'
[the full spec, restated cleanly: objective, files, interfaces,
constraints, verification command.]

PLANNING PHASE — you are sandboxed read-only and cannot write files.
Read whatever you need, then produce a concrete implementation plan:
the exact files to change, the shape of each change, and the order.

End your final message with exactly one verdict line:
  SPEC-OK
  SPEC-PROBLEM: <one-line reason>

Use SPEC-PROBLEM only for a real defect — the spec is ambiguous in a way
that changes the outcome, contradicts the code, or asks for something
architecturally unsound. A spec you can execute gets SPEC-OK.
SPEC_EOF
```

Run Sol, capturing both the plan text and the thread id:

```bash
# Portable timeout: macOS has no `timeout` unless coreutils is installed
T=$(command -v gtimeout || command -v timeout || true)
[ -z "$T" ] && echo "WARN: no timeout binary — codex runs uncapped (brew install coreutils to cap)"

${T:+$T 600} codex exec \
  --model gpt-5.6-sol \
  -c model_reasoning_effort=high \
  --sandbox read-only \
  --skip-git-repo-check \
  --cd "$(pwd)" \
  --json \
  --output-last-message "$PLAN" \
  - < "$SPEC" > "$EVENTS" 2>&1

TID=$(grep -o '"thread_id":"[^"]*"' "$EVENTS" | head -1 | cut -d'"' -f4)
```

`--sandbox read-only` is the enforcement mechanism, not a suggestion: Sol *cannot* write code, so this phase cannot quietly become implementation. `--json` is the only way to obtain `thread_id`.

## The plan gate

Read `"$PLAN"`. If its verdict line is `SPEC-PROBLEM`, **stop. Do not run phase 2.** Return:

```
CODEX REPORT
STATUS: spec-gap
MODELS: sol/high (plan) — halted before implementation
OBJECTIVE: [restated in one line]
PLAN: [Sol's plan, summarized]
PROBLEM: [Sol's SPEC-PROBLEM reason, verbatim]
GAPS: [what the caller must decide to unblock this]
```

The fix decision belongs to the caller, not to you and not to Terra. Do not "interpret" an ambiguous spec into a guess — that is how a cheap lane builds the wrong thing confidently. If the spec is wrong at the architectural level, the caller should consult `fable-advisor`.

On `SPEC-OK`, continue to phase 2.

## Phase 2 — implement (Terra, medium, workspace-write)

Terra resumes Sol's thread, inheriting the plan:

```bash
${T:+$T 600} codex exec resume "$TID" \
  --model gpt-5.6-terra \
  -c model_reasoning_effort=medium \
  -c sandbox_mode=workspace-write \
  --skip-git-repo-check \
  --output-last-message "$FINAL" \
  "IMPLEMENTATION PHASE. You now have workspace-write access — the read-only
restriction from the planning phase is lifted. Execute the plan you just
produced. Then run the verification command from the spec and include its
actual output in your final message."
```

`codex exec resume` accepts a **narrower flag set** than `codex exec` — there is no `--sandbox` and no `--cd`. The sandbox must be set with `-c sandbox_mode=workspace-write`, and the working directory is inherited from the shell, so you must already be at the repository root. `--sandbox` here fails with a usage error.

**If `$TID` is empty** (the thread id could not be extracted), fall back to re-briefing Terra in a fresh invocation with Sol's plan pasted in. This is still two-phase — Terra just reads the plan instead of remembering it:

```bash
BRIEF=$(mktemp -t codex-brief.XXXXXX)
{ cat "$SPEC"
  echo
  echo "=== IMPLEMENTATION PHASE ==="
  echo "The planning phase produced the plan below. Execute it, then run the"
  echo "verification command and include its actual output in your final message."
  echo
  echo "=== PLAN ==="
  cat "$PLAN"
} > "$BRIEF"

${T:+$T 600} codex exec \
  --model gpt-5.6-terra \
  -c model_reasoning_effort=medium \
  --sandbox workspace-write \
  --skip-git-repo-check \
  --cd "$(pwd)" \
  --output-last-message "$FINAL" \
  - < "$BRIEF"
```

You **must** say so in the report's `MODELS:` line when this fallback fires. Never let it silently degrade into a single-phase run.

## Flag discipline (non-negotiable)

| Flag | Why |
|---|---|
| `--sandbox read-only` (phase 1) | Sol plans. It cannot write, so planning cannot become implementation. |
| `-c model_reasoning_effort=high` (phase 1) | Sol is here for judgment — the plan is the expensive artifact. |
| `--json` (phase 1) | The only source of `thread_id`. Without it phase 2 cannot resume. |
| `-c sandbox_mode=workspace-write` (phase 2) | `resume` has no `--sandbox` flag. Codex writes code, scoped to the working tree. Never `danger-full-access`. |
| `-c model_reasoning_effort=medium` (phase 2) | Terra does volume, not judgment. The thinking already happened. |
| `--skip-git-repo-check` | Works outside git repos. |
| `--cd "$(pwd)"` (phase 1 / fallback only) | Deterministic working root. `resume` has no `--cd`; it inherits the shell's cwd. |
| `- < spec file` | Prompt via stdin. No quoting hazards, no truncated specs. |
| `${T:+$T 600}` | Ten-minute wall clock per phase when `timeout`/`gtimeout` exists (macOS needs `brew install coreutils`); uncapped otherwise. On timeout, report `STATUS: timeout` with whatever landed. |

The model slugs are documented defaults, not constants — if the caller's spec names different codex models for either phase, use those instead and say so in `MODELS:`.

## Verify independently

Read the diff (`git diff` / `git status`), run the spec's verification command **yourself**, and read Terra's final message from `"$FINAL"`. Codex's claim of success is not evidence; your re-run is.

## What you return

```
CODEX REPORT
STATUS: complete | partial | spec-gap | timeout | unavailable
MODELS: sol/high (plan) -> terra/medium (implement)
OBJECTIVE: [restated in one line]
PLAN: [one-line summary of Sol's plan]
CHANGES: [file — one-line summary, per file, from the actual diff]
VERIFIED: [verification command you re-ran — actual output evidence]
CODEX SAID: [one-line summary of Terra's final message, note any disagreement with the diff]
GAPS: [spec ambiguities, unfinished items, or "none"]
```

`MODELS:` exists so that any substitution is visible to the caller. If the resume failed and Terra was re-briefed, write `sol/high (plan) -> terra/medium (re-briefed, resume failed)`. If the caller overrode a model, name what you actually ran.

## Rules

- One two-phase run per task unless the caller explicitly decomposed it.
- Never claim completion without re-running the verification yourself. "Codex said it works" is forbidden as evidence.
- Never skip phase 1. The plan is what makes the cheap implementation phase safe.
- If Terra's changes are wrong, report that plainly with the failing output — do not patch them yourself. Fix decisions belong to the caller.
- If the task turns out to be architectural — the spec itself is wrong — that is a `spec-gap`, and it belongs upstream (consult `fable-advisor`).
````

- [ ] **Step 2: Verify the structural invariants**

Run:

```bash
cd ~/source/fable-advisor
echo "--- resume must NOT carry --sandbox or --cd ---"
grep -A6 'codex exec resume' agents/codex-implementer.md | grep -E '\-\-sandbox|\-\-cd' && echo "FAIL: forbidden flag on resume" || echo "OK"
echo "--- both models present ---"
grep -c 'gpt-5.6-sol' agents/codex-implementer.md
grep -c 'gpt-5.6-terra' agents/codex-implementer.md
echo "--- gate status present ---"
grep -q 'spec-gap' agents/codex-implementer.md && echo "OK" || echo "FAIL"
```

Expected: `OK`, then counts ≥1 for each model, then `OK`.

- [ ] **Step 3: Commit**

```bash
cd ~/source/fable-advisor
git add agents/codex-implementer.md
git commit -m "feat: two-phase codex lane — sol/high plans, terra/medium implements"
```

---

### Task 2: Delete the Grok lane and rewrite the orchestration doctrine

**Files:**
- Delete: `agents/grok-implementer.md`
- Modify: `skills/orchestration/SKILL.md` (lane table, deciding rule, parallelism section, cost-discipline line naming Sonnet)

**Interfaces:**
- Consumes: the `codex-implementer` report format from Task 1 (`spec-gap` status).

- [ ] **Step 1: Delete the Grok agent**

```bash
cd ~/source/fable-advisor
git rm agents/grok-implementer.md
```

- [ ] **Step 2: Fix the cost-discipline line that names Sonnet**

In `skills/orchestration/SKILL.md`, the "Cost discipline" section currently reads "spend Fable on judgment, spend Sonnet on volume" — Sonnet is no longer a lane. Replace that sentence:

- Find: `The whole economic case for this pattern is keeping its token volume low: spend Fable on judgment, spend Sonnet on volume. Three rules follow.`
- Replace with: `The whole economic case for this pattern is keeping its token volume low: spend Fable on judgment, spend the codex lane on volume. Three rules follow.`

- [ ] **Step 3: Replace the lane table and deciding rule**

Replace the entire `## The lanes` section (from the `| Lane |` table through the "Grok vs codex" paragraph and the re-route paragraph) with:

```markdown
| Lane | Producer | Invoke | Route here when |
|---|---|---|---|
| Implementation | GPT-5.6 Sol (plan) → GPT-5.6 Terra (implement) | `codex-implementer` agent | All well-specified implementation work. **The only implementation lane.** Requires the [Codex CLI](https://github.com/openai/codex). |
| Judgment | Fable 5 | `fable-advisor` agent | Not an implementation lane. See "Commitment boundaries" below. |

The implementation lane is itself two-phase: Sol at high reasoning reads the code and produces a plan, then Terra at medium reasoning inherits that plan and writes the code. The architect's cost discipline applied one level down — reason once with the expensive model, let the cheap one carry the volume.

Deciding rule: how much does the outcome depend on judgment the spec can't capture? Little → delegate; you will verify anyway. A lot, and mistakes are costly → keep it with the architect, or settle the decision with `fable-advisor` first and *then* delegate the settled spec.

The lane is a different model family than the architect, so every diff gets genuine cross-vendor review for free: OpenAI produces, Claude judges.

The lane's planning phase is also a spec check. If it returns `STATUS: spec-gap`, Sol judged the spec defective and refused to build against it — that is a signal to fix the spec (or consult `fable-advisor`), never to re-send the same spec with firmer wording.

If the lane returns `unavailable` or `timeout`, say so explicitly in your report — never quietly absorb a substitution. If the codex CLI is unavailable entirely, implement with a Claude subagent and state the downgrade plainly.
```

- [ ] **Step 4: Replace the parallelism section**

The current `## Parallelism` section ends with a two-lane race that no longer exists. Replace the whole section with:

```markdown
## Parallelism

Independent specs (no shared files, no ordering dependency) launch as parallel `codex-implementer` agents in a single message. Sequential chains and single-file surgery stay serial. Each lane invocation uses `mktemp` paths, so parallel runs do not collide.

For high-stakes work there is no second lane to race against — buy confidence with `fable-advisor` on the decision *before* delegating, and with your own diff review after.
```

- [ ] **Step 5: Verify no Grok references survive**

Run:

```bash
cd ~/source/fable-advisor
grep -ril 'grok' --exclude-dir=.git --exclude-dir=docs . || echo "OK: no grok references outside docs/"
ls agents/
```

Expected: `OK: no grok references outside docs/`, and `agents/` lists only `codex-implementer.md` and `fable-advisor.md`.

- [ ] **Step 6: Commit**

```bash
cd ~/source/fable-advisor
git add -A skills/ agents/
git commit -m "feat!: remove grok lane, codex is the sole implementation lane"
```

---

### Task 3: Update metadata and README for 4.0.0

**Files:**
- Modify: `.claude-plugin/plugin.json` (version, description, keywords)
- Modify: `.claude-plugin/marketplace.json` (metadata + plugin description)
- Modify: `README.md` (lane table, install URLs, requirements, use-it text, FAQ)

- [ ] **Step 1: `plugin.json`**

Set `version` to `4.0.0`. Replace `description` with:

> `Run your session on Claude's most capable model as a full-time architect: it writes specs and verdicts while a two-phase codex lane — GPT-5.6 Sol plans, GPT-5.6 Terra implements — does the typing. Fable-level judgment with cross-vendor implementation.`

Remove `"grok"` from `keywords`.

- [ ] **Step 2: `marketplace.json`**

Replace `metadata.description` with:

> `Model-routing patterns for Claude Code: architect-as-orchestrator with a two-phase GPT-5.6 (Sol plans, Terra implements) codex lane and a Fable 5 commitment-boundary advisor.`

Replace the plugin `description` with:

> `Architect-as-orchestrator: your session runs Claude's most capable model, routing implementation to a two-phase codex lane (GPT-5.6 Sol plans at high reasoning, GPT-5.6 Terra implements) — with a commitment-boundary advisor.`

- [ ] **Step 3: `README.md`**

Make these edits:

1. **Lane table** (lines ~7-11) → two rows:

```markdown
| Lane | Producer | Invocation | Route here when |
|---|---|---|---|
| Implementation | **GPT-5.6 Sol** (plans) → **GPT-5.6 Terra** (implements) | `codex-implementer` agent | All well-specified work — a two-phase codex run does the typing |
| Judgment | Fable 5 | `fable-advisor` agent | Commitment boundaries — see below |
```

2. **The paragraph after the table** → replace the Grok/racing prose with:

> Tokens route by volume: the expensive model emits the fewest tokens (judgment and specs), the cheap lane emits the most (code). The lane splits that again internally — Sol at high reasoning plans once, Terra at medium reasoning writes the code — so the premium is spent on thinking, not typing. Every implementation comes from a *different model family* than the architect that reviews it: cross-vendor review is built into the routing, not bolted on.

3. **Install block** → point at the fork:

```
claude plugin marketplace add jrajasekera/fable-advisor
claude plugin install fable-advisor@fable-advisor
```

4. **Requirements** → delete the Grok bullet entirely. Rewrite the codex bullet as required, not optional:

> - **Codex lane (the implementer):** the `codex-implementer` agent needs the [OpenAI Codex CLI](https://github.com/openai/codex) installed and authenticated (`npm i -g @openai/codex`, then `codex login`). It runs two phases: **GPT-5.6 Sol** (`gpt-5.6-sol`, `model_reasoning_effort=high`, read-only) plans, then **GPT-5.6 Terra** (`gpt-5.6-terra`, `model_reasoning_effort=medium`, workspace-write) implements by resuming Sol's thread. Without CLI access or model access, the agent reports `STATUS: unavailable` — it never silently falls back to a Claude model.

Also fix the parenthetical in the "Heads-up" bullet: `(This quiet fallback applies only to Claude model pins — the grok and codex lanes always fail loudly with a structured error.)` → `(This quiet fallback applies only to Claude model pins — the codex lane always fails loudly with a structured error.)`

5. **"Use it" section** → the sentence recommending a race must go:

> The architect writes the spec, delegates to `codex-implementer` (Sol plans it, Terra builds it), reads the diff and verification evidence when the report comes back, and only then reports done.

6. **FAQ** → rewrite two entries:

> **Upgrading from v3?** v4 removes the `grok-implementer` lane and makes `codex-implementer` the sole implementer, now running two phases: GPT-5.6 Sol plans at high reasoning, GPT-5.6 Terra implements at medium. The `fable-advisor` agent and advisor-only mode work exactly as before.

> **Why a GPT lane in a Claude plugin?** Vendor diversity. Models from one family share blind spots; an independent implementation from a different lineage catches what same-family review misses — and with Claude as the architect, *every* diff gets cross-vendor review for free. The architect stays Claude — the lane is a producer, not a judge.

- [ ] **Step 4: Verify**

```bash
cd ~/source/fable-advisor
jq -e '.version == "4.0.0"' .claude-plugin/plugin.json && echo "version OK"
jq -e '.keywords | index("grok") | not' .claude-plugin/plugin.json && echo "keywords OK"
jq empty .claude-plugin/marketplace.json && echo "marketplace.json parses"
grep -ril 'grok' --exclude-dir=.git --exclude-dir=docs . || echo "OK: no grok references outside docs/"
```

Expected: `version OK`, `keywords OK`, `marketplace.json parses`, `OK: no grok references outside docs/`.

- [ ] **Step 5: Commit**

```bash
cd ~/source/fable-advisor
git add .claude-plugin/ README.md
git commit -m "docs: 4.0.0 — two-phase codex lane, fork install URLs"
```

---

### Task 4: Repoint the Claude Code marketplace at the fork

**Files:** none in the repo — this mutates `~/.claude/plugins/`.

This must happen before Task 5, because the live end-to-end test dispatches the agent *as installed*.

- [ ] **Step 1: Push the branch and merge to main**

The marketplace installs from a git ref, so the changes must be on `main` of the fork.

```bash
cd ~/source/fable-advisor
git push -u origin codex-two-phase-lane
git checkout main
git merge --ff-only codex-two-phase-lane
git push origin main
```

- [ ] **Step 2: Remove the upstream marketplace, add the fork**

```bash
claude plugin marketplace remove fable-advisor
claude plugin marketplace add jrajasekera/fable-advisor
claude plugin install fable-advisor@fable-advisor
```

- [ ] **Step 3: Verify the installed copy is the fork's 4.0.0**

```bash
jq -r '.version' ~/.claude/plugins/cache/fable-advisor/fable-advisor/*/.claude-plugin/plugin.json 2>/dev/null
ls ~/.claude/plugins/cache/fable-advisor/fable-advisor/*/agents/
git -C ~/.claude/plugins/marketplaces/fable-advisor remote -v | head -1
```

Expected: `4.0.0`; agents dir contains only `codex-implementer.md` and `fable-advisor.md` (no `grok-implementer.md`); remote is `jrajasekera/fable-advisor`.

- [ ] **Step 4: Confirm Claude Code sees the agents**

Restart the session (or `/plugin`), then confirm `codex-implementer` appears in the agent list and `grok-implementer` does not.

---

### Task 5: Live end-to-end test of the two-phase lane

**Files:** none in the repo. This runs against a throwaway git repo in the scratchpad.

This is the real test. Structural greps prove the file says the right words; only this proves the lane *works*.

- [ ] **Step 1: Build a throwaway repo with a real, verifiable task**

```bash
SCRATCH=/private/tmp/claude-501/-Users-jrajasekera/b3ec6a5c-7fa6-4f16-a39f-39dc8cbf8b82/scratchpad/lane-test
rm -rf "$SCRATCH" && mkdir -p "$SCRATCH" && cd "$SCRATCH"
git init -q
cat > slugify.py <<'EOF'
def slugify(text):
    raise NotImplementedError
EOF
cat > test_slugify.py <<'EOF'
from slugify import slugify

def test_basic():
    assert slugify("Hello World") == "hello-world"

def test_punctuation_and_repeats():
    assert slugify("  Hello,   World!! ") == "hello-world"

def test_empty():
    assert slugify("") == ""
EOF
git add -A && git commit -qm "failing slugify"
python3 -m pytest test_slugify.py -q 2>&1 | tail -2
```

Expected: tests FAIL (`NotImplementedError`). This is the failing test the lane must make pass.

- [ ] **Step 2: Dispatch the real agent with a five-part spec**

Dispatch the `codex-implementer` agent (via the Agent tool) with this spec — pointing it at `$SCRATCH`:

```
Objective: Implement slugify() so the existing test suite passes.
Files: modify slugify.py in <SCRATCH path>
Interfaces: slugify(text: str) -> str
Constraints: standard library only; do not modify test_slugify.py
Verification: cd <SCRATCH path> && python3 -m pytest test_slugify.py -q
```

- [ ] **Step 3: Assert on the report and the evidence**

The returned `CODEX REPORT` must satisfy all of:

- `STATUS: complete`
- `MODELS: sol/high (plan) -> terra/medium (implement)` — **not** `re-briefed`. If it says re-briefed, the `thread_id` extraction is broken; fix the `grep -o '"thread_id":"[^"]*"'` line in the agent before proceeding.
- `PLAN:` is non-empty
- `VERIFIED:` contains actual pytest output showing 3 passed

Then confirm independently:

```bash
cd "$SCRATCH"
python3 -m pytest test_slugify.py -q 2>&1 | tail -2
git diff --stat
```

Expected: `3 passed`, and the diff touches only `slugify.py`.

- [ ] **Step 4: Test the plan gate fires on a bad spec**

Dispatch `codex-implementer` again with a deliberately unsound spec against the same repo:

```
Objective: Make slugify() faster by caching results in a global dict, and also
           change its return type to a list of tokens.
Files: slugify.py in <SCRATCH path>
Interfaces: unspecified
Constraints: none
Verification: cd <SCRATCH path> && python3 -m pytest test_slugify.py -q
```

This contradicts the existing tests (which require a hyphenated string, not a list). Sol should catch it.

Expected report: `STATUS: spec-gap`, with `PROBLEM:` naming the contradiction with the test suite, and **no changes written to the working tree**. Confirm:

```bash
cd "$SCRATCH" && git status --short
```

Expected: clean (the gate must halt before Terra runs).

If Sol instead returns `SPEC-OK` and Terra breaks the tests, the gate prompt in phase 1 is too weak — strengthen the SPEC-PROBLEM criteria in `agents/codex-implementer.md` and re-run.

- [ ] **Step 5: Record the evidence**

No commit here — this task produces evidence, not artifacts. Report both runs' `MODELS:` lines and the pytest output to the user. A lane that has not run is not done.

---

### Task 6: Finish the branch

- [ ] **Step 1: Confirm main is current**

If Task 4 Step 1 already merged to main, verify:

```bash
cd ~/source/fable-advisor
git checkout main
git log --oneline -5
git status --short
```

Expected: main contains all four commits, clean tree.

- [ ] **Step 2: Delete the merged branch**

```bash
cd ~/source/fable-advisor
git branch -d codex-two-phase-lane
git push origin --delete codex-two-phase-lane
```

- [ ] **Step 3: Tag the release**

```bash
cd ~/source/fable-advisor
git tag -a v4.0.0 -m "v4.0.0 — codex is the sole lane, split into sol/plan + terra/implement"
git push origin v4.0.0
```

---

## Self-Review

**Spec coverage:**
- Fork + repoint → Task 4. Version 4.0.0 → Task 3.
- Delete Grok lane (agent, doctrine, metadata, README) → Tasks 2 and 3.
- Racing-pattern removal → Task 2, Steps 3–4.
- Two-phase agent, plan gate, `thread_id` fallback, no-silent-fallback, per-phase timeout → Task 1.
- Report format with `MODELS:` / `PLAN:` / `spec-gap` → Task 1, asserted live in Task 5.
- Verified CLI facts (`read-only` phase 1, `-c sandbox_mode` on resume, no `--cd` on resume, `--json` for thread id) → Global Constraints + Task 1 flag table, asserted in Task 1 Step 2.
- Non-goals (don't touch `fable-advisor.md`, no Claude lane, no per-task approval) → respected; no task touches the advisor.

**Placeholder scan:** none. Every edit shows the literal replacement text; the agent rewrite is given in full.

**Type consistency:** the report fields (`STATUS`, `MODELS`, `PLAN`, `PROBLEM`, `GAPS`) and the status vocabulary (`complete | partial | spec-gap | timeout | unavailable`) are defined in Task 1 and referenced identically in Tasks 2 and 5. Model slugs and reasoning efforts match the Global Constraints throughout.

**Ordering:** Task 4 (install) must precede Task 5 (live test), because the test dispatches the agent as installed. Task 4 therefore merges to main mid-plan; Task 6 only cleans up.
