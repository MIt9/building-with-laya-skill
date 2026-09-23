# Building with Laya

This repo holds an agent skill for writing and improving programs that call **Laya** and **`laya-cli`** — Convai's local, non-generative System-1 decision model (Choice/Score/Noul). It is the Laya counterpart to [`dbreunig/building-with-jev-skill`](https://github.com/dbreunig/building-with-jev-skill) (TypeSafe Jev `jev-1.13`).

The skill covers question design (Choice, Score, Noul), state structure, answer composition in code, confidence thresholds and calibration, `laya-cli` usage (`predict` / `classify` / `filter` / `serve` daemon / `evaluate` / `shortlist` / `Router`), and diagnosis of questions that answer wrong or with low confidence. It targets the current Laya checkpoints (`convaiinnovations/laya`, `laya-multilingual`, `laya-typed-decisions`) and `laya-cli` `0.2.x`.

The skill lives in [`plugins/laya/skills/laya/SKILL.md`](plugins/laya/skills/laya/SKILL.md). The standalone CLI it wraps is [`MIt9/laya-cli`](https://github.com/MIt9/laya-cli) (`uv tool install laya-cli`).

This repo also ships:

- [`laya-browser-use`](plugins/laya-browser-use/skills/laya-browser-use/SKILL.md): action-heavy browser automation (navigation, clicks, toggles, scrolling) driven by local Laya through a Computer Use runtime — a fork of [`wy-coliney/jev-browser-use`](https://github.com/wy-coliney/jev-browser-use) with the cloud Jev API swapped for a local `laya-cli predict` call, so it needs no API key and makes no network call for decisions.
- [`subagents`](plugins/subagents/skills/subagents/SKILL.md): orchestrator/subagent coordination for Claude Code's own Agent tool (fork vs named `subagent_type`, `isolation`, `ListAgents`/`SendMessage`) — a fork of [`dobroslavradosavljevic/skills`](https://github.com/dobroslavradosavljevic/skills)' `subagents` skill, adapted to this harness's actual tools and its "only spawn when asked" default.
- [`laya-model-routing`](plugins/laya-model-routing/skills/laya-model-routing/SKILL.md): pick the cheapest model good enough for a task by asking local Laya its difficulty/kind/risk in one request — a fork of [`kerpopule/hermes-jev-skills`](https://github.com/kerpopule/hermes-jev-skills)' `jev-model-routing` skill, stripped of the Hermes fleet plugin and its catalog/pricing tooling down to the host-agnostic idea.

This repo is a [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) — each skill lives in its own `plugins/<name>/` folder with its own `plugin.json`, listed in [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json).

## Install

### Claude Code plugin

Run these two commands inside Claude Code:

```
/plugin marketplace add MIt9/building-with-laya-skill
/plugin install laya@building-with-laya
/plugin install laya-browser-use@building-with-laya   # optional, needs Computer Use
/plugin install subagents@building-with-laya           # optional, no Laya dependency
/plugin install laya-model-routing@building-with-laya  # optional
```

*Replace `MIt9/building-with-laya-skill` with your fork if you forked the repo.*

### Skills CLI

The [skills CLI](https://github.com/vercel-labs/skills) installs the skill into Claude Code, Codex, Cursor, and other agents:

```
npx skills add MIt9/building-with-laya-skill
```

### Manual

Copy the skill folder into your personal skills directory:

```
git clone https://github.com/MIt9/building-with-laya-skill
cp -r building-with-laya-skill/plugins/laya/skills/laya ~/.claude/skills/laya
```

## Use

Claude loads the skill on its own when a task involves Laya, `laya-cli`, typed decisions, or a System-1 classifier (routing, triage, guardrails, moderation, scoring). You can also invoke it by name with `/laya`.

**When to use `laya-cli` vs Python SDK:**

- **CLI (`laya-cli`)** — for shell pipelines, batch JSONL, quick human checks, and the resident daemon (`serve`) that avoids 10–35 s `laya.load()` on repeated calls:

  ```bash
  laya-cli predict "Refund the duplicate" --preset triage --format json
  cat candidates.jsonl | laya-cli classify --questions q.json | laya-cli filter --where "on_topic>=0.4" --sort -on_topic
  laya-cli serve --device mps & laya-cli predict "hello" --preset guard  # via daemon
  ```

- **Python SDK (`pip install laya`)** — for in-process services: `laya.load`, `Router(preload=True)`, `predict_shortlist`.

The skill explains both, with the same question-design rules for either surface.

## Comparison with Jev

|  | Jev (`jev-1.13`, TypeSafe API) | Laya (`laya-cli` / `laya` Python) |
|---|---|---|
| Hosting | Cloud API, closed weights | Local, Apache-2.0, `convaiinnovations/laya` on HF |
| State budget | 64k tokens shared, state + longest question ≤32k | 512 (English) / 1024 (multilingual) total; `head_max_len` 192/256 per question |
| Primitives | Choice / Score / Noul (same names) | Choice / Score / Noul (same names, `noul` may carry `criteria`) |
| Confidence | per-answer `confidence` + `probabilities` | `confidence` = 1 − normalized entropy; `noul` uses `max(p,1-p)` |
| Batch | All questions in one request (parallel) | Same — one forward pass per `predict` call |
| Large Choice | up to 255 options out of the box | ~20 options before `head_max_len` truncation — use `laya-cli --shortlist-k 20` / `laya.predict_shortlist` |
| CLI | `typesafe_sdk` + `TypeSafeClient` | `laya-cli` (`uv tool install laya-cli`) with `predict`/`classify`/`serve` daemon |

If you are porting a Jev program, keep the 7-step workflow and the diagnosis table — only the state budget, the `head_max_len` shortlist, and the `laya-cli serve` daemon are Laya-specific.

## Sources

- Laya upstream: https://github.com/NandhaKishorM/laya
- `laya-cli` (this skill's CLI): https://github.com/MIt9/laya-cli
- This skill adapts the structure of [`dbreunig/building-with-jev-skill`](https://github.com/dbreunig/building-with-jev-skill) `skills/jev/SKILL.md` (workflow, primitives, criteria, diagnosis, checklist) for Laya.
- `laya-browser-use` is a fork of [`wy-coliney/jev-browser-use`](https://github.com/wy-coliney/jev-browser-use) (MIT) with the cloud Jev/TypeSafe transport replaced by a local `laya-cli predict` subprocess call.
- `subagents` is a fork of [`dobroslavradosavljevic/skills`](https://github.com/dobroslavradosavljevic/skills) (MIT), retargeted from generic/proactive dispatch guidance to Claude Code's actual Agent tool and its conservative default.
- `laya-model-routing` is a fork of [`kerpopule/hermes-jev-skills`](https://github.com/kerpopule/hermes-jev-skills) (MIT) `jev-model-routing`, with the Hermes plugin and fleet-pricing tooling dropped down to the portable Laya-based routing idea.
