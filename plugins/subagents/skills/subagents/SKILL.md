---
name: subagents
description: Orchestrator/subagent coordination workflow for tasks that split into clear disjoint work, using Claude Code's Agent tool (fork vs named subagent_type, isolation:"worktree"/"remote", ListAgents, SendMessage to continue). Use only when the user explicitly asks to use subagents, says "use subagents", "spawn agents", "parallelize this", "delegate this", "split this across agents", or names one of the available agent types. This harness does not spawn subagents on its own initiative — do not use this skill to talk yourself into spawning when the user didn't ask.
---

# Subagents

## Overview

Coordinate work between the main session (orchestrator) and one or more subagents dispatched through the Agent tool. The orchestrator owns decomposition, conflict control, integration, verification, and final judgment; subagents explore or execute independent lanes.

This harness's default stance is conservative: it does not spawn subagents unless the user explicitly asks, or the request names a specific agent type. A task having "multiple angles" or being "thorough" is not, by itself, a request to spawn — handle it inline first. This skill governs *how* to run a dispatch once one is warranted, not a license to decide on your own that a task would benefit from one.

## Core Rule

Use subagents only when the user has asked for them (directly, or by naming an agent type) **and** they make the work safer, faster, or more thorough than doing it inline.

Before dispatching, decide whether the task has clear disjoint lanes. If the task is small, tightly coupled, or likely to cause file conflicts, skip subagents and say why — even if the user asked, a trivial task doesn't need parallel dispatch; do the work and note that decomposition wasn't worth it.

## Choosing how to dispatch

The Agent tool has three shapes; picking the right one is most of getting this right:

- **`subagent_type: "fork"`** — inherits the orchestrator's full conversation context and runs on the same model. Use it for research/investigation lanes where the subagent needs everything discussed so far and you don't want its raw tool output (search results, file dumps) filling the orchestrator's own context. Forks share the orchestrator's prompt cache, so they're cheap.
- **A named `subagent_type`** (or none, for general-purpose) — starts cold with zero memory of this conversation. Use it for a bounded, well-specified lane a smart colleague could execute from a self-contained brief: state the goal, scope, and constraints explicitly, because nothing here is implied. Pick the type whose tool access matches the lane (a read-only investigation lane gets `Explore`, not a full-access type).
- **`isolation: "worktree"`** — gives the subagent its own git worktree, so it can edit files without racing the orchestrator's or another subagent's edits to the same tree. This is the concrete fix for "two lanes would need to edit the same files": don't split by hoping they stay disjoint, give the risky one its own worktree. `isolation: "remote"` runs in a cloud environment instead, for work that shouldn't touch the local machine at all.

Never fabricate or predict a dispatched agent's findings before its result actually arrives. A forked or backgrounded agent's result lands later as its own notification — if the user asks about progress before then, report that it's still running, not a guess at what it will say.

## Orchestrator Responsibilities

- Understand the user request and inspect enough context to split work safely.
- Identify independent lanes by file ownership, subsystem, question, artifact, or review concern.
- Avoid assigning overlapping writes to multiple subagents; use `isolation: "worktree"` where edits could collide instead of hoping they won't.
- Give each subagent a bounded prompt with scope, allowed files or areas, expected output, and constraints. For a non-fork agent, write it like a brief to a colleague who just walked in — explain the goal, what's already been tried, and why it matters, not just a terse instruction.
- Keep secrets, credentials, hidden reasoning, and unrelated user changes out of subagent prompts.
- Track each subagent's status, findings, outputs, and risks. `TaskCreate`/`TaskUpdate`/`TaskGet`/`TaskList`/`TaskOutput` hold this when the lane is tracked as a task; `ListAgents` shows what's currently running or reachable, and `SendMessage` resumes a specific agent (by name) with more context instead of starting a fresh one that re-derives everything.
- Integrate results centrally instead of letting subagents make conflicting final decisions.
- Run final verification and produce the final answer.
- Trust but verify: a subagent's summary describes what it intended to do, not necessarily what it did. When it wrote or edited code, check the actual diff before reporting the lane done.

## Good Subagent Lanes

Dispatch subagents for lanes such as:

- Researching separate libraries, APIs, docs, or product options.
- Auditing different concerns, such as correctness, security, UX, performance, docs, or tests.
- Inspecting different code areas that do not require simultaneous edits to the same files.
- Implementing isolated modules, tests, fixtures, docs, or examples with clear boundaries — give each its own `isolation: "worktree"` if their edits could otherwise touch the same files.
- Reproducing independent failures or collecting logs from separate systems.

Prefer one subagent per meaningful lane. Do not create extra agents just to appear parallel — each spawn re-derives context you may already have; a fork is cheap because it shares your cache, a fresh agent is not.

## Bad Subagent Lanes

Do not dispatch subagents when:

- The user didn't ask for subagents and the task doesn't clearly need them — handle it inline.
- The task is trivial enough that coordination overhead is larger than the work.
- The work requires one continuous design judgment from a single editor.
- Multiple agents would need to edit the same files or closely coupled code at the same time and no `isolation: "worktree"` boundary separates them.
- The repo is in a fragile dirty state and parallel edits would risk overwriting user work.
- The available subagent's result can't return enough detail to verify or integrate safely (a narrow read-only type, when the lane actually needs judgment).
- The user asks for direct implementation without delegation and the task is narrow.

## Dispatch Prompt Checklist

Each subagent prompt should include:

- The user goal.
- The exact lane assigned to the subagent.
- Files, directories, systems, or sources the subagent should inspect.
- Files or areas the subagent may edit, if editing is allowed.
- Files or areas the subagent must not touch.
- Expected deliverable: findings, patch summary, test result, recommendation, or artifact.
- Verification expectations for that lane.
- A request to report blockers and uncertainty explicitly.
- For a non-fork agent: enough background that it doesn't have to guess — it has no memory of this conversation. For a fork: what's already been tried, so it doesn't redo it.

When the lane is research or review, ask for evidence and actionable findings rather than broad commentary. Prescribed steps become dead weight when the lane's premise is wrong — hand over the question, not a rigid procedure, unless the task genuinely needs one followed exactly.

## Integration Workflow

1. Confirm the user actually asked for subagents (directly or by naming a type); if not, do the work inline instead.
2. Create a short decomposition plan.
3. Decide whether subagents are useful and safe for the identified lanes.
4. If useful, dispatch bounded lanes — in parallel, in one message with multiple Agent calls, when the harness supports it and the lanes are truly independent.
5. Review each subagent result before trusting it; a summary is a claim, not a verified fact.
6. Resolve contradictions, duplicates, and scope drift.
7. Apply or integrate accepted changes centrally, preserving unrelated user work.
8. Run verification that covers the combined result.
9. Summarize which subagents were used, what each handled, what changed, and what verification passed.

## Conflict Control

- Treat the orchestrator as the only final integrator.
- Prefer subagents that produce findings or focused patches over broad uncontrolled edits.
- If two lanes unexpectedly overlap, stop parallel editing for that overlap and merge manually.
- Re-read files before applying or integrating work when another agent may have touched nearby code.
- Never overwrite user changes or another agent's work blindly.
- When edit collision risk is foreseeable rather than surprising, prevent it upfront with `isolation: "worktree"` instead of relying on lane discipline alone.

## Final Response

Report:

- Whether subagents were used, and why (the user asked, or named a type).
- If used, the lanes assigned, the dispatch shape (fork / named type / isolation), and what each returned.
- If skipped, the reason.
- Final integrated changes or conclusions.
- Verification performed and any remaining risks.

## Sources

- Adapted from [`dobroslavradosavljevic/skills`](https://github.com/dobroslavradosavljevic/skills) `skills/subagents/SKILL.md` (MIT). This fork replaces the generic "prefer subagents for harder tasks" trigger and the OpenAI-specific `agents/openai.yaml` interface descriptor with this harness's actual Agent tool semantics (fork vs named `subagent_type`, `isolation`, `ListAgents`/`SendMessage`, `TaskCreate`/`TaskUpdate`) and its conservative default of only spawning when asked.
