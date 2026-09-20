# Jev Skill — Open-Source Typed AI Decision Harness

Jev Skill is an open-source typed AI decision harness for Jev, TypeSafe, and
local LLMs. It turns unstructured model guesses into validated Noul, Choice,
and Score decisions with consensus, verification, calibration, abstention,
and CI gates.

[![Tests](https://github.com/aiwithenoch/Jev-Skill/actions/workflows/test.yml/badge.svg)](https://github.com/aiwithenoch/Jev-Skill/actions/workflows/test.yml)
[![ClawHub](https://img.shields.io/badge/ClawHub-Jev%20Skill-6f42c1)](https://clawhub.ai/aiwithenoch/skills/jev-skill)

Open-source repository: <https://github.com/aiwithenoch/Jev-Skill>

Built for developers shipping reliable AI agents, LLM routing, RAG systems,
structured extraction, model evaluation, and production automation.

## Built by AI With Enoch

Jev Skill is created and maintained by Enoch Ansong, an AI engineer and
creator from Ghana, founder of Riverside AI and AI With Enoch. The project is
part of a wider mission to build practical, accessible AI systems and help
more people across Africa create with AI.

- Website: <https://aiwithenoch.com>
- GitHub: <https://github.com/aiwithenoch>
- LinkedIn: <https://www.linkedin.com/in/blacctbwoy>
- X/Twitter: <https://x.com/aiwithenoch>
- Instagram: <https://www.instagram.com/aiwithenoch>
- TikTok: <https://www.tiktok.com/@aiwithenoch>

## Join the community

Share benchmarks, adapters, golden cases, failure examples, and production
lessons through the [AI With Enoch community](https://github.com/aiwithenoch/aiwithenoch-community).
Community preview work also lives in
[ai-with-enoch-community](https://github.com/aiwithenoch/ai-with-enoch-community).
Issues and pull requests are welcome here—especially improvements that make
local AI more reliable and understandable.

## Why this exists

Most model integrations ask for prose and then hope the answer is correct.
That makes routing, grading, extraction, and automation fragile. Jev Skill
defines the decision explicitly, validates the shape of every response, tests
the model on reviewed cases, and escalates uncertainty instead of hiding it.

Jev Skill treats a model as a semantic decision layer: the application owns
schemas, thresholds, arithmetic, policy, side effects, and escalation. The
included harness lets the same Noul/Choice/Score question set run against:

- TypeSafe Jev (`POST /v1/systemone`)
- OpenJev's local TypeSafe-compatible server (`POST /v1/systemone`) backed by
  one-pass option scoring on Apple Silicon/MLX
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

Run against the community [OpenJev](https://github.com/daseinlabs/open-jev)
server after its Apple-Silicon setup and `make serve`:

```bash
git clone https://github.com/daseinlabs/open-jev.git
cd open-jev
make setup
make serve
```

Then, from the Jev Skill repository, run:

```bash
python3 scripts/jev_harness.py \
  --provider openjev \
  --base-url http://127.0.0.1:8000 \
  --model jev-local \
  --questions questions.json \
  --cases cases.jsonl \
  --concurrency 1 \
  --output openjev-report.json
```

OpenJev is a community implementation, not TypeSafe's proprietary Jev
weights. Its native endpoint returns the same typed response surface, so this
repo can measure its accuracy, calibration, latency, and review rate beside
the hosted model without changing the question set. OpenJev's MLX scorer is
deterministic; repeated samples are useful for transport stability, not
sampling diversity.
OpenJev's default zero-shot log-probability scores are not automatically
calibrated; use a labeled golden set and the harness calibration metrics before
using thresholds in production.

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

## FAQ

### Does Jev Skill require the Jev API?

No. It can run against OpenJev, Ollama, vLLM, LM Studio, or llama.cpp server.
The Jev API gives you Jev's model; local providers give you the same typed
contract and reliability harness using the model you run.

### Does it make a local model as intelligent as Jev?

No. It cannot change model weights. It makes local-model decisions more
structured, testable, measurable, and safer to deploy.

### Can Codex or Claude Code use it?

Yes. They can create question sets, generate golden cases, run the harness,
review reports, and help fix failures. The harness itself calls model
endpoints and does not depend on an interactive coding agent.

### What does “Noul / Choice / Score” mean?

Noul answers a yes/no question with a probability. Choice selects one named
category. Score returns a bounded, ordered level with a probability
distribution. Your application still owns the final policy and side effects.

## Roadmap

- More provider adapters and reproducible local benchmark packs.
- Calibration fitting and threshold selection for domain-specific costs.
- Shared adversarial and metamorphic case libraries.
- First-class integrations for AI coding agents and CI platforms.

## Research and design notes

See [SKILL.md](SKILL.md), [references/harness.md](references/harness.md), and
[references/research-notes.md](references/research-notes.md). The notes link
the official TypeSafe docs, local structured-output APIs, evaluation
frameworks, self-consistency/verifier research, and calibration references.

## License

MIT-0. See [LICENSE](LICENSE).
