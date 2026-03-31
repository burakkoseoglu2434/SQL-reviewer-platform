# FEATURES: SQL Agent Değerlendirme Platformu

## Table Stakes (Must Have)
- Offline dataset comparison format (Golden Case).
- AST / Structural Validator Rule Generator.
- LLM As A Judge Component with Confidence Scores.
- Exact Results / Metric diff verification via Sandbox execution DB.

## Differentiators
- Composite Scoring calculation (%30 rule, %25 exec, %25 LLM vs).
- "Answer Grounding" test: verifying user's text output vs standard output.
- Dedicated Human Review / Annotation Panel logic for boundary scores (0.4 - 0.6).

## Anti-Features
- Auto-finetuning pipelines (it makes evaluation systems biased and tightly coupled to the base model immediately).
