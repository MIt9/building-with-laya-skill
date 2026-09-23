---
name: laya
description: Write and improve programs that call Laya or laya-cli — local, typed decisions with Choice/Score/Noul. Use when designing Laya questions, structuring state, composing answers in code, setting confidence thresholds, running laya-cli predict/classify/serve/filter, or diagnosing a Laya question that answers wrong or with low confidence.
---

# Building with Laya (and laya-cli)

Laya is a local, non-generative judgment model. It reads one `state` (text, or a JSON object like `{"subject": "...", "body": "..."}`) and answers every question in the request independently and in parallel, in a single forward pass — ~33 ms for one question, ~7 ms/q batched (T4), 20–35 ms on a laptop GPU. It never generates text. `laya-cli` (`pip install laya-cli` / `uv tool install laya-cli`) is the ergonomic CLI wrapper around the same checkpoints (`convaiinnovations/laya`, `laya-multilingual`, `laya-typed-decisions`).

Use Laya when code needs a **decision**, not a sentence: which queue, is this spam, how severe, should the agent escalate. Do not use it when the task needs multi-step reasoning, arithmetic, or free-form extraction — keep the LLM and put Laya in front as a cheap System-1 first pass.

This guidance targets the current `laya` checkpoints (English 512 ctx, multilingual/typed-decisions 1024 ctx) and `laya-cli` `0.2.x` (`predict` / `classify` / `filter` / `serve` / `evaluate`).

## Workflow

1. List the decisions your code must make. Write each as a branch, a threshold, or a ranking.
2. Write one question per judgment. Split any question that weighs two properties.
3. Pick the primitive whose answer your code acts on directly.
4. Build the smallest state that answers every question. Compute in code whatever code can compute.
5. Put every question that shares the state into one request, including questions that matter for only some inputs. One forward pass = all answers.
6. Combine the answers in code: branches, weights, and confidence gates.
7. Test against labeled examples. Read `probabilities` / `noul` on the misses, then revise one or two questions at a time.

## Choose the primitive

| Primitive | Use it when | Returns | Code acts on it with |
| --- | --- | --- | --- |
| Choice | Answer is one of a known set, no order | `choice`, `probabilities` per option, `confidence` | branch per `choice` |
| Score | Answer is a position on a spectrum you can describe in steps | `score` (expected level, float), `probabilities` per level, `legend`, `confidence` | threshold / rank / weight |
| Noul | Answer is a clean yes/no and the probability is the signal | `noul` = P(true) 0→1, `confidence` = max(p,1-p) | `if p >= threshold` |

- Add an `other` / `none of the above` option to a Choice whose list may not cover every input.
- A Noul of 0.5 means unsure — not "medium". Use a Score for degree.
- A Noul needs a crisp condition. "Is this candidate strong in Python?" is vague. "Does the resume state that the candidate used Python at work?" is crisp.
- For Laya `noul` you may pass `criteria: {"true": "phishing, scam", "false": "legitimate"}` when the boundary is subtle.
- Laya `confidence` is `1 - normalized entropy` of the distribution (peaked = high). It is low when probability is spread, even if the top option is right. `noul` has no separate `confidence` field — use distance from 0.5 or `max(p,1-p)`.

## Write the instructions

- State the exact condition. Laya answers the words you wrote.
- Ask one property per question. Hidden second judgments lower accuracy and confidence.
- Name the part of the state the question judges, with a backticked path: `` `ticket.messages[0].text` `` or `` `body` ``. Laya is an encoder doing textual entailment — the path helps it locate the signal.
- Write directly. Avoid double negatives, property-of-property, and multi-hop.
- Keep numerals that stand for levels out of the instructions. "Rate 0 to 2" gives nothing to match.
- Write the full question in `instructions`. The question ID never reaches the model.
- Keep decision policy out of the question. "A shared address cannot override a name conflict" belongs in code.
- **Ask what the text says, not what to do about it.** ` "Where is the bird relative to the gap?"` beats `"Which way must the bird move?"` — the latter inverts because `"up"` is pulled toward `"above"` in the state. Ask a perception question, let code map to action.
- When you explain what you really meant after a wrong answer, that explanation is the missing half of the instruction. Add it.

Instructions may be a string, object, or array. Use an object when the question has labeled parts:

```json
"instructions": {
  "question": "Does the `message` ask the recipient to disclose a sensitive credential?",
  "inspect": "message",
  "focus": "Look for a request to send the credential itself, not a request to change or reset it."
}
```

Useful keys: `question`, `focus`, `inspect`, `note`, `compare` (list of state paths), `field` (`name`, `type`, `unit`, `description`).

## Write the criteria

Treat criteria as an extension of the instruction — they must ask for the same thing, in the same direction.

**Choice.** Map each option to a description. Make them contrastive when options are close:

```json
"billing": {
  "what": "Charges, invoices, refunds, subscriptions",
  "not_for": "Order tracking or account access",
  "examples": ["I was charged twice", "Where is my refund?"]
}
```

A bare `["billing", "technical"]` works, but `{"billing": "invoices, payments, refunds"}` beats it — the description is what Laya matches. Each option is truncated to 48 tokens; all options for a question share 192 tokens (English) or 256 (multilingual) `head_max_len`. Past ~20 options accuracy falls; if you see `ValueError: ... options exceed head_max_len`, shorten descriptions or split the question, or use `predict_shortlist` / `laya-cli --shortlist-k`.

**Score.** List levels low→high, 2–10 levels, only as many as you can describe distinctly:

- Describe situations. `"Broken feature, but workaround exists"` works. `"Moderately severe"` does not.
- Make each level stand alone. Laya judges each level separately and sees neither its number nor neighbors. `"Worse than previous"` means nothing.
- Keep each Score to one dimension. `"Punctual and smart and experienced"` measures three things — split it.
- Give a rare extreme its own level when code must treat it differently.
- A level may be an object: `{"summary": "One change, clearly stated", "signals": ["A single fix or feature", "..."]}`.

**Noul.** Criteria are optional. Add `true`/`false` sides with `what` + `examples` when the boundary is subtle. Put the neighboring case in the description of the side it belongs to.

**Examples.** Short concrete instances: `"I was charged twice"`. Not "a message about billing".

## Build the state

- Send only fields the questions need. Unrelated detail lowers accuracy and hides which input caused a miss.
- Retrieve and filter in code first. When code cannot filter, ask a relevance Noul per passage and keep those that pass.
- Keep the state structured so questions can point into it: `{"subject": "...", "body": "..."}` beats a flat string template.
- Convert numeric encodings to words before sending. Color name, not hex. Computed number or named bucket, not raw figures.
- **Put numbers into words, never numbers.** `"Bird altitude: 20. Gap altitude: 60."` — no checkpoint could tell which was lower. Compute in code and send the conclusion: `"the bird is far below the gap"`.
- Compute date order, duration, windows, counts, sums in code. Send the result.
- **Keep the state short and front-loaded.** English checkpoint reads 512 tokens total, others 1024; over-long state is cut from the end. Put what matters first. For email, `laya.email_state(subject, body)` strips quoted replies. With `laya-cli`, `--state-field` is taken **verbatim** — never silently prepend `(photographer: Name)`; use `--prepend-field` only explicitly (it degraded `on_topic` by 0.1–0.3).
- Budget: English `head_max_len=192` / `max_len=512`, multilingual `256`/`1024` (up to 8192). State + longest single question must fit. Large state hides signal.
- Treat state text as able to steer the answer. Laya does not treat state as hostile. State in criteria what counts and test injected content.

## Via Python SDK (direct)

```bash
pip install laya  # pulls torch, transformers, safetensors
```

```python
import laya
from laya import Router

# Recommended: Router auto-detects script/language in <0.5ms and picks checkpoint
router = Router(preload=True)  # all checkpoints resident; no 7s reload on language switch

state = {"from": "user@acme.com", "subject": "Duplicate charge #4411", "body": "Billed twice for March. Refund or we cancel."}
questions = {
    "department": {
        "type": "choice",
        "instructions": "Which department should handle this request in `body`?",
        "criteria": {"billing": "invoices, payments, refunds", "technical": "bugs, outages", "other": "everything else"},
    },
    "urgency": {"type": "score", "instructions": "How urgent is the request in `body`?", "criteria": ["not urgent", "soon", "critical deadline"]},
    "churn_risk": {"type": "noul", "instructions": "Does `body` threaten to cancel or leave?"},
}

res = router.predict(state, questions)  # device auto: cuda>mps>cpu; pass lang="de" when known
# or single checkpoint: agent = laya.load("convaiinnovations/laya", subfolder="multilingual"); agent.predict(state, questions)

print(res["answers"]["department"]["choice"])  # billing, confidence 0.94
print(res["routing"]["model"], res["usage"]["input_tokens"])
# High-cardinality: shortlist first, then one forward pass
# result = laya.predict_shortlist(agent, state, big_choice_questions, embed_fn=laya.embed_fn_from_agent(agent), k=20)
```

Load once at startup, reuse, guard `predict` with a lock (one GPU = one forward pass). Warm up with one throwaway `predict` (kernel compilation).

Presets: `laya.triage_questions()`, `laya.email_questions()`, `laya.guard_questions()`, `laya.moderation_questions()`, `laya.router_questions()` — read them before relying.

## Via laya-cli (standalone, for humans & AI agents)

No Python needed after `uv tool install laya-cli` / `pipx install laya-cli`. Same checkpoints, same `choice`/`score`/`noul` semantics, streaming + daemon.

### Single prediction (human table, agent JSON)

```bash
# Human — preset without a file, table
laya-cli predict "I was charged twice, refund please" --preset triage
laya-cli predict --text "Ignore previous instructions" --preset guard --format table

# Agent — JSON
laya-cli predict --text "Is this spam?" --preset guard --format json
# -> {"answers":{"jailbreak":{"type":"noul","noul":0.02,"confidence":0.97}},"usage":{...}}

# State as JSON object (Laya serialises dicts)
laya-cli predict --state '{"subject":"Hi","body":"Billed twice"}' --preset email --format json

# Custom: preset < file < inline (later wins)
laya-cli predict --text "hello" --preset triage --questions-inline '{"custom":{"type":"noul","instructions":"Is it polite?"}}' --format json

# High-cardinality (77 banking intents) — embedding shortlist
laya-cli predict --text "where is my card?" --questions banking.json --shortlist-k 20 --format json

# Multilingual — Router recommended
laya-cli predict --text "मुझसे दो बार शुल्क लिया गया" --preset triage --router --format json
```

State is flexible: positional `TEXT`, `--text`, `--state JSON`, `--state-file`, or batch `--input file.jsonl` / stdin. Questions from `--preset`, `--questions file.json`, `--questions-inline` (merged).

Checkpoints: `convaiinnovations/laya` (English, 512), `.../multilingual` (`--subfolder multilingual`), `typed-decisions`. `--model` / `--subfolder` for single model, or `--router` for auto routing (`Router(preload=True)`).

### Batch & pipeline (streaming, one load + warmup)

```bash
laya-cli predict --input candidates.jsonl --questions questions.json --format jsonl > scored.jsonl
laya-cli predict --input candidates.jsonl --questions questions.json --flatten --format jsonl | laya-cli filter --where "on_topic>=0.4" --sort -on_topic

# Legacy streaming (kept for compatibility, e.g. pexels-cli px)
cat candidates.jsonl | laya-cli classify --questions questions.json | laya-cli filter --where "on_topic>=0.4" --sort -on_topic
px videos --queries "..." --state --dedupe keep-first | laya-cli classify --questions questions.json | laya-cli filter --where "on_topic>=0.4" --sort -on_topic
```

`--state-field` is verbatim. `filter` is a tiny post-filter: `--where "field>=0.4,other!=spam"` (ops `== != = > < >= <=`, comma=AND), `--sort "-on_topic,+id"` (`-` desc).

### Presets, info, evaluate

```bash
laya-cli questions list --format table
laya-cli questions triage > questions.json
laya-cli info  # python/torch/cuda/mps + HF cache
laya-cli evaluate --questions questions.json --labeled labeled.jsonl --label-field label --threshold 0.5
# -> accuracy per question, passing@threshold, precision@threshold, escalation rate
```

### Resident daemon (optional, for iterative tuning)

`laya.load()` costs 10-35 s (MPS). `laya-cli serve` keeps the model resident; subsequent `predict`/`classify`/`evaluate` with the **same config** (`model|subfolder|device|router|lang` → `~/.cache/laya-cli/daemons/<hash>.json`, `device` resolved `cuda>mps>cpu` before hashing) hit `127.0.0.1` and skip the load (<1 ms).

```bash
laya-cli serve --device mps &                 # background, writes pid+port file
laya-cli serve status                         # GET /status → {model,device,loaded_at,idle_seconds,requests_served,pid,port}
laya-cli predict "hello" --preset guard --format json  # via daemon, no load
laya-cli predict "hello" --preset guard --no-daemon --format json  # force in-process
laya-cli serve stop; laya-cli serve stop --all # graceful POST /shutdown or SIGTERM
# HTTP also: POST /predict {"state":..., "questions":{...}}, GET /status, POST /shutdown — only 127.0.0.1, Lock-serialized
```

`--idle-timeout 1800` default, `0` disables. Warmup throwaway `predict` before serving. If no daemon, commands silently fall back to in-process — no pipeline change.

## Compose the answers in code

Speculative fan-out: ask every question your code might need in one request, including questions that matter only on some branches — they run in parallel, little extra latency. Second request only when code cannot build it without the first answer.

Confidence-gated routing: answer says what, confidence says whether to act. Set a floor (e.g. 0.6) below which no action runs, and a per-action threshold that rises with cost of being wrong. Three paths: act / confirm-or-flag / hand off. For `noul`, threshold the `noul` probability itself; for `choice`/`score`, threshold `confidence` or the top `probabilities` value.

Composite scoring: split a complex judgment into one Score per dimension, normalize each `score / (len(criteria)-1)`, combine with weights in code. Change a weight, not the question.

Intent routing: Choice for intent + Score for complexity; route each intent to deterministic code, a specialist LLM, or a person; send low-confidence to a person.

Taxonomy walk / counting / dates / extraction: same patterns as Jev — one Choice per tree level, one Noul per item then sum, one Choice per date part then assemble in code, regex/LLM generates candidates then Choice/Noul verifies.

```python
from laya import Router
router = Router(preload=True)
res = router.predict(
    {"ticket": ticket_text},
    {
        "category": {"type": "choice", "instructions": "Which category fits the main request in `ticket`?", "criteria": {"bug_report": "Something is broken", "billing": "Charges, invoices, refunds", "other": "Anything else"}},
        "severity": {"type": "score", "instructions": "How severe is the issue reported in `ticket`?", "criteria": ["Cosmetic", "Broken with workaround", "Blocking, no workaround"]},
        "refund_requested": {"type": "noul", "instructions": "Does `ticket` explicitly ask for a refund or credit?"},
    },
)
if res["answers"]["category"]["confidence"] < 0.6:
    route_to_human(ticket_id)
elif res["answers"]["category"]["choice"] == "bug_report":
    norm = res["answers"]["severity"]["score"] / 2
    if norm > 0.75 and res["answers"]["severity"]["confidence"] > 0.5: escalate(ticket_id)
elif res["answers"]["category"]["choice"] == "billing" and res["answers"]["refund_requested"]["noul"] > 0.7:
    route_to_billing(ticket_id)
```

With `laya-cli`:

```bash
# One file: questions.json with category (choice), severity (score), refund_requested (noul)
cat tickets.jsonl | laya-cli predict --questions questions.json --format jsonl > scored.jsonl
# In code (Python/Node), read scored.jsonl, branch on .answers[category].choice / .confidence
```

## Read the answers

- `score` is probability-weighted mean of levels. `1.0` can mean certainty on `1` or split `0`+`2` — read `probabilities`.
- Threshold a score, rank by it, or round it — do not interpolate a quantity. Levels are weakly calibrated as numbers.
- `confidence` is `1 - normalized entropy` — peaked = high. F841. For `noul`, use `max(p,1-p)` or distance from 0.5. Every answer stays inside the options you supplied.
- `usage.input_tokens` reports tokens consumed. `answers[*].action.act_probability` is the act-or-escalate head.
- `laya-cli predict --format json` returns `{answers, usage, routing?}`; `classify --flatten` expands to `{qid, qid_p, qid_confidence, qid_probs?, qid_score}`.

## Improve a program

Find the failing question before changing anything. Collect labeled examples, run them — with `laya-cli evaluate --questions q.json --labeled labeled.jsonl` or `agent.predict` in a loop — and compare `probabilities`/`noul` against labels. Change one or two questions per revision.

| Symptom | Likely cause | Fix |
|---|---|---|
| Wrong answers, high confidence | Instruction taken literally | State the exact condition. Put the boundary case in criteria. |
| Low confidence on Choice | Options overlap or no option fits | Add `what`/`not_for`/`examples`. Add `other`. |
| Low confidence on Score | Levels overlap, two dimensions, or missing state | Rewrite levels as distinct situations. Split question. Add field to state. |
| Scores cluster middle | Levels are degrees/numbers | Describe concrete situations per level. Remove numerals. |
| Top-of-scale cases look alike | Extreme has no level | Add a level for the extreme. |
| Noul hovers ~0.5 | Vague condition | Define condition. Add `true`/`false` criteria with examples. |
| Accuracy falls as inputs grow | State has irrelevant detail | Filter in code. Send only needed fields. |
| Errors on counts/sums/dates/numeric nearness | Laya doing arithmetic | Move arithmetic to code. Ask only per-item Noul or extraction. |
| Nested/negated questions | Too much indirection | Ask direct question. Name state path. Split into two and combine in code. |
| Answer follows state text | Content steers model | Tighten criteria. Test adversarial. Gate on confidence. |
| Rewording trades one error for another | One question weighs several properties | Split into atomic questions, combine in code. |
| Final decision wrong while each answer right | Policy wrong | Change weights/thresholds in code, not questions. |
| Slow / costly | Questions spread over sequential calls | Merge into one request. Second request only when it depends on first. Use `laya-cli serve` for iterative tuning. |

Rules: judge a revision on labeled data (higher confidence alone is not better); keep answer space stable once code depends on it; write general rules in instructions, examples only for instances.

## Checklist

- [ ] Each question asks one property and a person could answer it in a second
- [ ] Primitive matches how code uses the answer (`choice`→branch, `score`→threshold, `noul`→if)
- [ ] Instructions state the exact condition and name state paths in backticks
- [ ] Criteria agree with instructions and point the same direction; each `choice` option has a description
- [ ] Score levels describe situations, stand alone, carry no numerals, and are low→high
- [ ] Choices that may not cover every input have an `other` option
- [ ] State holds only what the questions need, front-loaded, fits 512/1024 tokens, and numeric comparisons are done in code
- [ ] All questions on the same state travel in one request; speculative fan-out included
- [ ] Every action has a confidence threshold matched to its risk, and low confidence has a fallback
- [ ] Weights/thresholds live in code, not in questions; `laya-cli --no-daemon` reproduces in-process for debugging
- [ ] `head_max_len` (192/256) respected — no question exceeds it; large Choice uses `--shortlist-k`
- [ ] Labeled examples back every revision (`laya-cli evaluate`); `shortlist` and `router` tested separately

## Sources

- Laya upstream: https://github.com/NandhaKishorM/laya, checkpoints `convaiinnovations/laya` (English 512), `laya-multilingual` (1024, 322M), `laya-typed-decisions`; `laya.predict_shortlist` + `embed_fn_from_agent` for high-cardinality
- This repo's CLI: `laya-cli` (`uv tool install laya-cli`, `src/laya_cli/cli.py:1`, `src/laya_cli/daemon.py:1`) — `predict`/`classify`/`filter`/`serve`/`evaluate`/`questions`/`info`
- Laya integration skill (local): `~/.claude/skills/laya-integration/SKILL.md` — device, warmup, Router, calibration, and sidecar pattern that `serve` implements
- Jev patterns this skill adapts: confidence-gated routing, composite scoring, intent routing, taxonomy walk, counting, dates, extraction (see `building-with-jev-skill` `skills/jev/SKILL.md` structure)
