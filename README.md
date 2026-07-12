# Fable Advisor

**The smartest model runs the show. Cheaper models do the typing.**

Claude Code lets every subagent run on a different model — and lets the session itself run on a different model than its subagents. This plugin exploits that with the **architect pattern**: your session runs on **Fable 5**, Anthropic's most capable model, acting as a full-time architect. It owns requirements, decomposition, specs, and verification — and routes every implementation task to the cheapest adequate lane:

| Lane | Producer | Invocation | Route here when |
|---|---|---|---|
| Implementation | **GPT-5.6 Sol** (plans) → **GPT-5.6 Terra** (implements) | `codex-implementer` agent | All well-specified work — a two-phase codex run does the typing |
| Judgment | Fable 5 | `fable-advisor` agent | Commitment boundaries — see below |

Tokens route by volume: the expensive model emits the fewest tokens (judgment and specs), the cheap lane emits the most (code). The lane splits that again internally — Sol at high reasoning plans once, Terra at medium reasoning writes the code — so the premium is spent on thinking, not typing. Every implementation comes from a *different model family* than the architect that reviews it: cross-vendor review is built into the routing, not bolted on.

The plugin ships the **orchestration skill** — the routing doctrine that teaches the session when to use each lane, the cost discipline that keeps the expensive model's own token volume minimal (emit judgment not volume, keep context lean, reason once then hand off), the five-part spec contract that makes context-free delegation safe, and the verification rules that keep the cheap lane honest.

## Install

```
claude plugin marketplace add jrajasekera/fable-advisor
claude plugin install fable-advisor@fable-advisor
```

Updating an existing installation to the latest release:

```
claude plugin marketplace update fable-advisor
claude plugin update fable-advisor@fable-advisor
```

Then start your session as the architect:

```
/model fable
```

**Lite mode — one file, 30 seconds.** Don't want the full pattern? Copy [`agents/fable-advisor.md`](agents/fable-advisor.md) into `~/.claude/agents/` and keep your session on Sonnet. You get advisor consults at commitment boundaries without the orchestration layer (see "Advisor-only mode" below).

## Requirements

- **Claude Code ≥ 2.1.170** with a subscription that includes Fable 5 (Pro, Max, Team, or Enterprise — all current consumer plans qualify).
- **No Fable access** (e.g. API-key billing)? Use `/model opus` for the session and change `model: fable` → `model: opus` in the advisor file. Same pattern, model tiers shift down one.
- **Codex lane (the implementer):** the `codex-implementer` agent needs the [OpenAI Codex CLI](https://github.com/openai/codex) installed and authenticated (`npm i -g @openai/codex`, then `codex login`). It runs two phases: **GPT-5.6 Sol** (`gpt-5.6-sol`, `model_reasoning_effort=high`, read-only) plans, then **GPT-5.6 Terra** (`gpt-5.6-terra`, `model_reasoning_effort=medium`, workspace-write) implements by resuming Sol's thread. Without CLI access or model access, the agent reports `STATUS: unavailable` — it never silently falls back to a Claude model.
- Heads-up: if a pinned Claude model isn't available on your account, Claude Code silently falls back to your session model — the pattern degrades quietly rather than erroring. If results feel unremarkable, check your plan. (This quiet fallback applies only to Claude model pins — the codex lane always fails loudly with a structured error.)

Model resolution order in Claude Code: `CLAUDE_CODE_SUBAGENT_MODEL` env var → per-invocation `model` parameter → agent frontmatter → session model.

## Use it

With the session on Fable, just ask for work — the orchestration skill routes it:

```
Add rate limiting to our public API. Design it, delegate the
implementation, and verify the evidence before you call it done.
```

The architect writes the spec, delegates to `codex-implementer` (Sol plans it, Terra builds it), reads the diff and verification evidence when the report comes back, and only then reports done.

To make the doctrine always-on, add one line to your project's `CLAUDE.md`:

```
You are the architect running the most expensive model — minimize your
own token volume. Delegate all implementation through the orchestration
skill's routing table (never type code yourself), delegate broad codebase
exploration to cheap read-only agents, and verify evidence before
accepting any lane's report.
```

## Commitment boundaries

Even the architect gets a second opinion. The `fable-advisor` agent is a read-only skeptic — consulted before architecture decisions, migrations, API designs, and whenever a problem has resisted two attempts. It reads your actual code and returns a verdict in under 300 words. It never implements. Running it from a Fable session still pays: it sees the code fresh, without your conversation's accumulated assumptions.

## Advisor-only mode (the original pattern)

The inverse arrangement, for when you'd rather keep the session cheap: run the session on Sonnet and consult `fable-advisor` only at commitment boundaries.

```
Migrate our checkout sessions from Postgres to Redis — plan it,
consult your advisor before committing, then implement.
```

A typical consult costs cents. To make it automatic, add to your project's `CLAUDE.md`:

```
Before committing to any architecture decision, migration, or refactor
touching 3+ files, consult the fable-advisor agent and act on its verdict.
```

## FAQ

**Is this Anthropic's "advisor tool"?** No — that's a server-side API feature. These are plain Claude Code subagents plus a skill: readable, editable, no beta flags.

**Does this work on claude.ai?** No — subagent model routing is Claude Code only (CLI, desktop, VS Code, web).

**Why not just run everything on Fable?** You can. It's excellent. It's also the most expensive lane per token, and most of a session's tokens are implementation mechanics that the cheap lane handles at near-parity. Spend the premium where judgment lives.

**Upgrading from v3?** v4 removes the `grok-implementer` lane and makes `codex-implementer` the sole implementer, now running two phases: GPT-5.6 Sol plans at high reasoning, GPT-5.6 Terra implements at medium. The `fable-advisor` agent and advisor-only mode work exactly as before.

**Why a GPT lane in a Claude plugin?** Vendor diversity. Models from one family share blind spots; an independent implementation from a different lineage catches what same-family review misses — and with Claude as the architect, *every* diff gets cross-vendor review for free. The architect stays Claude — the lane is a producer, not a judge.

## Go deeper

I write [**Attention Heads**](https://attentionheads.substack.com/?utm_source=github&utm_medium=readme&utm_campaign=fable-advisor) — deep, evidence-backed writing on AI, cognition, and agentic engineering. The **Agentic Engineering Field Notes** series is where I publish practical advice on the craft of using AI. [Subscribe](https://attentionheads.substack.com/subscribe?utm_source=github&utm_medium=readme&utm_campaign=fable-advisor) to get new posts to your inbox.

## License

MIT
