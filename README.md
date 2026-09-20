# Jev Skill

Typed decision workflows and a dependency-free evaluation harness for Jev and
local models.

[![Tests](https://github.com/aiwithenoch/Jev-Skill/actions/workflows/test.yml/badge.svg)](https://github.com/aiwithenoch/Jev-Skill/actions/workflows/test.yml)
[![ClawHub](https://img.shields.io/badge/ClawHub-Jev%20Skill-6f42c1)](https://clawhub.ai/aiwithenoch/skills/jev-skill)

Open-source repository: <https://github.com/aiwithenoch/Jev-Skill>

Jev Skill treats a model as a semantic decision layer: the application owns
schemas, thresholds, arithmetic, policy, side effects, and escalation. The
included harness lets the same Noul/Choice/Score question set run against:

- TypeSafe Jev (`POST /v1/systemone`)
- Ollama native structured outputs (`POST /api/chat`)
- OpenAI-compatible local servers such as vLLM, LM Studio, and llama.cpp

The local path does not claim to turn a smaller checkpoint into Jev. It
reproduces the typed contract and adds strict validation, test-time consensus,
optional verification, explicit review/abstention, calibration, and CI gates.

## Quick start

Validate a question map and JSONL golden set without making a model call:

```bash
python3 scripts/jev_harness.py \
  --questions questions.json \
  --cases cases.jsonl \
  --validate-only
```

Run against Jev:

```bash
export TYPESAFE_API_KEY="..."
python3 scripts/jev_harness.py \
  --provider typesafe \
  --model jev-1.13.0 \
  --questions questions.json \
  --cases cases.jsonl \
  --output jev-report.json
```

Run against Ollama with a five-sample ensemble and verifier:

```bash
python3 scripts/jev_harness.py \
  --provider ollama \
  --base-url http://127.0.0.1:11434 \
  --model qwen3:8b \
  --samples 5 --temperature 0.4 --seed 17 --verify \
  --questions questions.json \
  --cases cases.jsonl \
  --output local-report.json
```

Run against vLLM, LM Studio, or llama.cpp server:

```bash
python3 scripts/jev_harness.py \
  --provider openai-compatible \
  --base-url http://127.0.0.1:8000/v1 \
  --model local-model \
  --samples 5 --temperature 0.4 --verify \
  --questions questions.json \
  --cases cases.jsonl
```

Use `--structured-protocol llama.cpp` for llama.cpp's direct schema request
shape. Structured output is strict by default; `--allow-json-repair` is an
explicit diagnostic escape hatch and should remain off in CI.

## Question and case format

Questions are a JSON object. A Noul is a yes/no probability, a Choice selects
one criterion key, and a Score uses an ordered level list:

```json
{
  "urgent": {
    "type": "noul",
    "instructions": "Does this require urgent attention?"
  },
  "team": {
    "type": "choice",
    "instructions": "Which team should handle this?",
    "criteria": {
      "billing": "Charges and refunds",
      "technical": "Bugs and outages",
      "other": "None of the above"
    }
  },
  "severity": {
    "type": "score",
    "instructions": "How severe is this?",
    "criteria": ["Minor", "Material", "Critical"]
  }
}
```

Cases are JSONL. `expected` is optional for collection-only runs:

```json
{"id":"ticket-1","tags":["golden","billing"],"state":{"message":"I was charged twice."},"expected":{"urgent":true,"team":"billing","severity":{"level":1,"tolerance":0.5}}}
```

## Reliability stack

The harness composes the following layers without asking the model for a chain
of thought:

1. schema-constrained JSON where the provider supports it;
2. strict typed response validation and finite/range checks;
3. independent sample aggregation over typed distributions;
4. consensus measurement and review routing;
5. an optional second typed verifier over evidence and candidate answers;
6. golden, metamorphic, adversarial, boundary, and context-rot slices;
7. Brier/log loss, calibration, Wilson accuracy intervals, latency, retries,
   tokens, optional cost, and CI quality gates;
8. state-size and warning-rate guardrails for production runs.

Consensus or verifier support never silently changes the proposed answer. Low
support becomes `decision: "review"`; application code decides whether that
means a human, deterministic checker, or fallback model.

## Development

The harness has no third-party Python dependencies. Run the checks from the
repository root:

```bash
python3 -m unittest discover -s tests -v
python3 -m py_compile scripts/jev_harness.py
python3 /Users/admin/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```

The tests use local fake HTTP servers and never require a TypeSafe key or a
running model. Live model benchmarks should be run separately with sensitive
golden sets kept out of version control.

## Research and design notes

See [SKILL.md](SKILL.md), [references/harness.md](references/harness.md), and
[references/research-notes.md](references/research-notes.md). The notes link
the official TypeSafe docs, local structured-output APIs, evaluation
frameworks, self-consistency/verifier research, and calibration references.

## License

MIT-0. See [LICENSE](LICENSE).
