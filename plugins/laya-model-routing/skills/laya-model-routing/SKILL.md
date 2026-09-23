---
name: laya-model-routing
description: Pick the cheapest model that's good enough for a task, using local Laya to score difficulty, kind, and risk in one fast request before delegating work, calling a cheaper model, or spawning a subagent. Use when choosing a model, cutting model spend, or deciding which model a delegated task should run on.
---

# Model routing with Laya

Ask before you assume. Laya reads a task description and answers, in one forward pass: how hard it is (Score: trivial/routine/substantial/expert), what kind of work it is (Choice: coding/writing/research/general), and whether a mistake would be costly (Noul). Code then walks a pool of models for that tier and specialty and picks the first one that fits (context size, vision requirement). This is a plain `laya-cli predict` call — it works in any harness, not tied to a particular fleet or plugin system.

## When to use it

- Before delegating a task to a cheaper or different model.
- Before spawning a subagent (see the [`subagents`](../../../subagents/skills/subagents/SKILL.md) skill in this marketplace) — route the lane's model before dispatch, not after the subagent has already run on the wrong one.
- When you want to cut spend without hand-guessing which tasks actually need a frontier model.

## Ask in one request

Laya's "speculative fan-out" rule applies here directly (see the base [`laya`](../../../laya/skills/laya/SKILL.md) skill): ask everything the routing decision needs in one request, not three round trips.

```bash
laya-cli predict --state '{"task": "<the task, in the requester'"'"'s own words>"}' --questions-inline '{
  "difficulty": {"type":"score","instructions":"How hard is the work described in `task`?","criteria":["Trivial or mechanical: a lookup, reformat, rename, short factual reply, or a single obvious step","Routine: ordinary multi-step work with a clear path and low ambiguity","Substantial: needs planning, several interacting parts, debugging, or careful judgment","Expert: subtle, ambiguous, or high-stakes — architecture, security, concurrency, data migration, legal or money"]},
  "kind": {"type":"choice","instructions":"What kind of work is described in `task`?","criteria":{"coding":"Writing, changing, debugging or reviewing software, scripts, configs or shell commands","writing":"Drafting or editing prose, marketing, messages, documents or creative text","research":"Finding, comparing or synthesizing information, analysis, or current events","general":"Conversation, planning, operations, or anything that is none of the others"}},
  "risky": {"type":"noul","instructions":"Would a mistake on `task` be costly — production, money, security, legal, or irreversible data loss?"}
}' --format json
```

## Turn the answers into a model

- `difficulty.score` (0-3, float): threshold it into a tier — below 0.5 simple, below 2.5 medium, otherwise hard. Read `difficulty.probabilities` on a boundary case before trusting the mean; a 1.4 split cleanly across "routine"/"substantial" is a different situation from a 1.4 that is actually two lumps at the extremes.
- Risk sets a floor, not a ceiling: if `risky.noul >= 0.6`, never resolve below `medium`, even when `difficulty` alone says simple. A short "delete the production database" prompt is exactly the case a difficulty-only score would miss.
- `kind.choice` selects the specialty pool (`coding` / `writing` / `research` / `general`). A missing specialty pool falls back to `general` — never invent a model for a specialty you haven't configured a pool for.
- Near a model's context limit, don't downgrade. Rebuilding a prompt cache on a cheaper model usually costs more than the routing decision saves.
- Low confidence, a malformed or refused answer, or `laya-cli` unreachable: keep whatever model the task was already going to use. Routing is an optimization, never a gate — it must not block or delay the task itself.

## Pools

A small JSON file naming which models are acceptable at each tier and specialty is enough for a single project or user — no catalog scraper or price-band generator needed until you're actually managing more models than you can list by hand. `~/.config/laya-model-routing/pools.json`:

```json
{
  "simple": {"general": ["provider:cheap-model"], "coding": ["provider:cheap-coding-model"]},
  "medium": {"general": ["provider:mid-model"],   "coding": ["provider:mid-coding-model"], "research": ["provider:mid-research-model"], "writing": ["provider:mid-writing-model"]},
  "hard":   {"general": ["provider:frontier-model"], "coding": ["provider:frontier-coding-model"]}
}
```

First entry in a pool wins; the order is your preference, not a ranking Laya produces. Add a model where you'd actually want a task of that shape to land, and nowhere else — an unused pool entry is dead weight the next reader has to puzzle over.

## Guarantees worth keeping

- Unsure is not hard: on a harmless task, an unsure difficulty answer keeps the current tier; on a task the risk Noul flags, unsure moves it to medium, never higher just from uncertainty.
- Judge a task on what it actually asks for, not on boilerplate wrapped around it. If you're routing a templated job (a cron task, a repeated report), extract the instruction from the template rather than scoring the whole envelope — the envelope is mostly not what should be scored.
- Don't send secrets or full sensitive payloads as `task` text. Describe the shape of the work ("refactor a payment-processing module", not the module's code with live keys in it) when the content itself is sensitive — Laya runs locally, but the state-hygiene discipline is the same one the base `laya` skill documents for any state you send it.
- Tune from logged decisions, not a hunch. If you log routing output, keep the JSON answer and drop the raw `task` text unless you've deliberately decided to keep it — small logged state ages better than a long-lived transcript of exactly what people asked.

## Sources

- Adapted from [`kerpopule/hermes-jev-skills`](https://github.com/kerpopule/hermes-jev-skills) `skills/jev-model-routing/SKILL.md` (MIT), which wires the same three-question idea into the Hermes fleet's `hermes-jev` plugin, its `/jev` commands, an OpenRouter-catalog price-band pool generator, and an escalation ladder for handing hard work to a frontier seat. This version keeps only the host-agnostic core — one Laya request scoring difficulty/kind/risk, code walks a pool — and drops the Hermes plugin, its automatic per-turn routing, and the fleet-scale catalog/pricing tooling; add those back only if you're actually operating a multi-model fleet that needs them.
