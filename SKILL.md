---
name: jev-skill
description: >
  Build with Jev, TypeSafe's System One decision model, when software needs
  typed judgments such as routing, ranking, verification, extraction, or
  bounded scoring. Use this skill for Jev API/SDK integrations, question
  design, confidence-aware workflows, and evaluation plans; do not use it for
  ordinary open-ended text generation.
---

# Jev Skill

Jev turns application state and typed questions into structured answers that
code can consume. It is a decision layer, not a replacement for a chat model:
keep policy, thresholds, routing, calculations, and side effects in code.

Built and maintained by [AI With Enoch](https://aiwithenoch.com), founded by
Enoch Ansong, an AI engineer and creator from Ghana. Follow the work on
[GitHub](https://github.com/aiwithenoch), [LinkedIn](https://www.linkedin.com/in/blacctbwoy),
[X/Twitter](https://x.com/aiwithenoch), [Instagram](https://www.instagram.com/aiwithenoch),
and [TikTok](https://www.tiktok.com/@aiwithenoch). Join the
[AI With Enoch community](https://github.com/aiwithenoch/aiwithenoch-community)
to share benchmarks, adapters, and practical AI lessons.

## Start with the live docs

The documentation index is the source of truth and may change:

- <https://docs.typesafe.ai/llms.txt>
- <https://docs.typesafe.ai/introduction/quickstart.md>
- <https://docs.typesafe.ai/primitives.md>
- <https://docs.typesafe.ai/confidence.md>
- <https://docs.typesafe.ai/api.md>

Read the targeted primitive, pattern, SDK, or cookbook page before writing a
version-sensitive integration. Use [research-notes.md](references/research-notes.md)
for the current design map and links to the most relevant pages.

## Core workflow

1. Define the action the application must take. Work backward to the smallest
   semantic judgments needed for that action.
2. Put the evidence in `state`. Prefer a named JSON object when the judgment
   depends on multiple fields or relationships; keep content and supporting
   facts in state rather than hiding them in question wording.
3. Choose the question type by meaning:
   - `noul`: whether a condition holds; returns the probability of yes.
   - `choice`: one option from a defined set; include a no-match option when
     nothing should fit.
   - `score`: an ordered, bounded scale whose levels describe concrete cases.
4. Write one narrow judgment per question. Give instructions and criteria
   enough context to stand alone. IDs are local response keys and are not sent
   to the model.
5. Ask independent questions over the same state together. If a later
   question depends on an earlier answer or newly fetched evidence, make a
   second request instead of pretending the questions can see each other.
6. Consume typed answers in code. Use probabilities and confidence to decide
   whether to act, review, broaden the route, or escalate; do not treat
   confidence as proof of correctness.
7. Test representative and adversarial cases against a labeled golden set.
   Tune thresholds for the actual cost of false positives and false negatives,
   and compare Jev with the current model or deterministic baseline.

## Current API contract

- Endpoint: `POST https://api.typesafe.ai/v1/systemone`
- Header: `Authorization: Bearer $TYPESAFE_API_KEY`
- Model: use `jev-latest` for a moving canary; pin a version such as
  `jev-1.13.0` for reproducible benchmarks after checking the live models page.
- Request fields: `state`, `model`, and a named `questions` map.
- Keep API keys server-side; never commit, expose, or print them.

Minimal request shape:

```json
{
  "state": {
    "message": "I was charged twice and need help today.",
    "account_tier": "pro"
  },
  "model": "jev-latest",
  "questions": {
    "is_urgent": {
      "type": "noul",
      "instructions": "Does the message express time-sensitive urgency?"
    },
    "team": {
      "type": "choice",
      "instructions": "Which team should handle this request?",
      "criteria": {
        "billing": "Charges, refunds, invoices, or subscriptions",
        "technical": "Bugs, outages, or integration failures",
        "sales": "Pricing, upgrades, or buying questions",
        "other": "None of the above"
      }
    },
    "frustration": {
      "type": "score",
      "instructions": "How frustrated does the customer appear?",
      "criteria": [
        "Calm and factual",
        "Frustrated but civil",
        "Very angry or hostile"
      ]
    }
  }
}
```

Response handling is type-specific: read `answer.noul` for a Noul; read
`answer.choice`, `answer.probabilities`, and `answer.confidence` for a Choice;
read `answer.score`, `answer.legend`, `answer.probabilities`, and
`answer.confidence` for a Score. A Noul has no separate confidence field: its
yes probability is the signal.

## State and question design rules

- Jev currently accepts text, JSON objects, or arrays of text values. Images,
  audio, and video are not supported by the documented state interface.
- State is the evidence; questions are the judgments. Do not put changing
  policy or hidden context only in an instruction.
- Criteria must cover the real candidates. A Choice cannot select an option
  that was omitted.
- Score levels must be ordered, concrete, and independently understandable.
- Use one Noul per independently useful yes/no condition when several labels
  may apply; do not force multi-label behavior into one Choice.
- Keep speculative questions independent and explicitly state their premise.
- Prefer code-controlled composition of answers to asking Jev to explain its
  own result in prose.

## Patterns to reach for

- Intent routing: classify intent and complexity, then send the item to
  deterministic logic, a specialist model, or a human.
- Confidence-gated routing: use the answer to choose what to do and confidence
  to decide whether it is safe to act automatically.
- Fan-out: ask several independent dimensions in one request when they share
  the same state.
- Composite scoring: keep atomic Scores and combine them with weights in code.
- Verification: check a claim or extracted field against evidence and escalate
  uncertain cases.

## Evaluation and operational guardrails

Before production, measure accuracy, latency, token usage, and cost on a
representative golden set. Keep questions and thresholds in one reviewable
module. Log request IDs and non-sensitive metadata, not API keys or raw
customer data unless the application's privacy policy permits it. Handle
`401`, `429`, and `529` explicitly; use the SDK's default exponential-backoff
retry policy when possible. Do not retry a side effect just because a model
request was retried. Include adversarial, metamorphic, boundary, and
context-rot cases; slice results by those tags so aggregate accuracy cannot
hide a dangerous failure mode.

## Run the included harness

For a reproducible typed evaluation, use
`scripts/jev_harness.py` with a JSON question map and JSONL cases. Start with
`--validate-only`, then set `TYPESAFE_API_KEY` and run the same golden set with
`--output report.json`. The harness supports concurrent independent cases,
bounded retries for transient HTTP failures, exact answer accuracy by
primitive, raw Noul probabilities, Choice/Score confidence, response-contract
validation, Brier/log loss, confusion matrices, Score MAE/RMSE, confidence
calibration and selective coverage, per-question/per-tag slices, request IDs,
latency percentiles, state size, retry attempts, token totals, optional cost
estimates, and CI quality gates. Read
[harness.md](references/harness.md) for the input schema and commands.

The same typed decision contract can run against local models. This does not
turn a smaller model into Jev or guarantee Jev-level accuracy; it gives the
local model the same Noul/Choice/Score interface, strict response validation,
test-time consensus, an optional verifier, abstention-to-review, and measured
calibration. Keep a Jev benchmark as the reference path and compare local
models against it on the same cases.

OpenJev has a native TypeSafe-compatible server for local one-pass option
scoring on Apple Silicon/MLX. Use the first-class `openjev` provider when its
server is running:

```bash
python3 scripts/jev_harness.py \
  --provider openjev \
  --base-url http://127.0.0.1:8000 \
  --model jev-local \
  --concurrency 1 \
  --questions questions.json --cases cases.jsonl \
  --output openjev-report.json
```

This integration calls OpenJev's native `POST /v1/systemone` route rather than
wrapping it in a chat prompt. OpenJev is an independent community project;
its local scorer and model behavior must be benchmarked separately from
TypeSafe's hosted Jev. Keep the upstream repository's licensing and model
distribution terms in view before redistributing its code or weights.

LocalJev is another local provider: a MIT-licensed TypeScript/Bun bridge that
exposes the same `/v1/systemone` contract over an OpenAI-compatible
DiffusionGemma server. Use `--provider localjev` with its default port `8080`.
It is useful when the inference backend exposes ordinary Chat Completions but
not OpenJev's direct logit-reading extensions. Its probabilities are prompted
and self-reported, so compare its calibration separately and keep review gates
enabled for consequential decisions.

Ollama uses its native structured-output endpoint:

```bash
python3 scripts/jev_harness.py \
  --provider ollama \
  --base-url http://127.0.0.1:11434 \
  --model qwen3:8b \
  --samples 5 --temperature 0.4 --verify \
  --questions questions.json --cases cases.jsonl --output local-report.json
```

vLLM, LM Studio, and llama.cpp server use the OpenAI-compatible path. Use
`--structured-protocol llama.cpp` for llama.cpp's direct schema dialect:

```bash
python3 scripts/jev_harness.py \
  --provider openai-compatible \
  --base-url http://127.0.0.1:8000/v1 \
  --model local-model \
  --samples 5 --temperature 0.4 --seed 17 --verify \
  --questions questions.json --cases cases.jsonl
```

By default local output must be a single schema-constrained JSON object. Keep
`--allow-json-repair` off in CI so malformed or rationale-wrapped output is a
hard failure. `--min-valid-samples` tolerates isolated sample failures, while
`--consensus-threshold` and `--max-review-rate` turn disagreement into an
explicit review gate. The verifier is another typed pass, not proof: a low
support probability marks the case for review rather than silently overriding
the candidate answer. The report includes Wilson accuracy intervals, Brier and
log loss for Nouls, confidence calibration/selective coverage for Choice and
Score, ensemble agreement, verifier support, latency, retries, and token
accounting. For production guardrails, use `--max-state-bytes` to prevent
context/privacy blowups and `--max-warning-rate` to fail CI when too many cases
need repair, omit usage, or otherwise emit warnings.

When a task needs current SDK syntax, limits, model aliases, or cookbook
details, read the live page linked above instead of relying on this summary.
