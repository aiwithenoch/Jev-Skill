# Jev Skill

## What It Does

This skill builds typed Jev decision workflows and evaluates them over
reviewed golden sets. It supports TypeSafe Jev plus local Ollama and
OpenAI-compatible runtimes such as vLLM, LM Studio, and llama.cpp server.
The included harness validates Noul, Choice, and Score answers; can aggregate
independent samples; can run a typed verifier; and marks uncertain cases for
review instead of silently forcing a decision.

## Best For

- Typed routing, ranking, extraction, verification, and bounded scoring.
- Comparing Jev with local models on the same golden, adversarial, and
  metamorphic cases.
- Measuring accuracy, calibration, consensus, latency, retries, token usage,
  and optional cost.
- Building a safe local-model decision layer with schema-constrained output
  and explicit abstention.

## Not For

- Claiming that a local checkpoint has Jev's weights or accuracy without a
  matched benchmark.
- Replacing deterministic arithmetic, date handling, permissions, or policy
  enforcement with model output.
- Treating confidence, consensus, or a verifier score as proof of truth.

## Safety Notes

- API keys come from environment variables and are never written to reports.
- Input state is not printed or stored in reports; use appropriate privacy
  controls for golden sets and model endpoints.
- Local output is strict JSON-schema validated by default; JSON repair is an
  explicit opt-in for diagnostics.
- Low consensus or verifier support becomes `decision: "review"`; the
  harness does not silently override the proposed answer.
