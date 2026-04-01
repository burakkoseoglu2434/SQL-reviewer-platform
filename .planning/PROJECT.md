# SQL Evaluation & LLM-as-a-Judge Platform

## What This Is

A Python CLI evaluation platform that takes JSON/CSV output from a SQL-generating agent and runs it through a four-layer judge pipeline (rule engine → execution validator → AST/semantic validator → LLM judge) against a curated golden dataset. It produces per-case composite scores with approve/rewrite/human_review/reject decisions, prints batch evaluation reports to the terminal, and tracks regression, hallucination, and observability metrics over time. Storage starts with CSV/JSON files and migrates to PostgreSQL for production.

## Core Value

Catch SQL agent errors in evaluation — not in production — by making quality measurable, traceable, and automatically regressionable against a live golden dataset.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Golden dataset ingestion from CSV (968-row `golden_eval_dataset_202603311349.csv`)
- [ ] Golden case schema: question, gold_sql, gold_answer, execution_metadata, business_context, bucket
- [ ] Agent output ingestion from JSON/CSV files (candidate_sql, answer, result)
- [ ] Rule Engine layer: structural checks, business rules, risk patterns (LIMIT, CURRENT_DATE, JOIN multiply, NULL)
- [ ] Execution Validator layer: execute candidate SQL against CSV-based sandbox; execution_accuracy, non_empty_execution_accuracy, subset_non_empty_execution_accuracy, routing_accuracy
- [ ] AST/Semantic Validator layer: parse SQL to AST, normalize, compare to gold SQL for semantic equivalence
- [ ] Result Set Comparator: row count, column count, column name (case-insensitive), numeric tolerance atol=1e-5, order independence
- [ ] LLM Judge layer: intent_fidelity, business_correctness, answer_grounding, holistic decision + rewrite guidance
- [ ] Judge Output Quality scorer: specificity, completeness, rationale_density, actionability
- [ ] Composite scoring: intent_fidelity 25%, execution 20%, semantic 15%, business_correctness 15%, result_equivalence 10%, answer_grounding 10%, risk 5%
- [ ] Critical override rules: execution error / unparseable SQL / critical hallucination → force reject
- [ ] Decision thresholds: ≥0.85 approve, 0.65–0.84 rewrite, 0.40–0.64 human_review, <0.40 reject
- [ ] Batch runner: replay all golden cases, produce per-case results
- [ ] CLI report: pass_rate, non_empty_execution_accuracy, valid_sql_rate, hallucination_rate, regression list, avg_latency, token_usage
- [ ] Hallucination tracking: answer hallucination + judge hallucination detection
- [ ] Observability metadata per run: model_name, latency_ms, input/output tokens, prompt_version, execution_time_ms, sql_ast_hash
- [ ] Structured output per case: decision, overall_score, per-dimension scores, issues list (type/severity/evidence), rewrite guidance
- [ ] PostgreSQL storage layer (agent_replay_runs, judge_scores, judge_issues, hallucination_events, monthly_reports)

### Out of Scope

- Real-time online evaluation / shadow mode — planned for Phase 5, not v1
- Web dashboard / UI — CLI output only for v1
- Offline fine-tuning / model retraining — evaluation pipeline only, not training loop
- Multi-tenant access control — single internal team
- OAuth / authentication — internal tool

## Context

- **Golden dataset**: `golden_eval_dataset_202603311349.csv` (968 rows) already exists in workspace; needs schema enrichment (execution_metadata, business_context, bucket classification) as part of Phase 0
- **Existing SQL agent**: Python-based, full access to modify; outputs JSON/CSV files with candidate_sql, answer, and result fields
- **Users**: ML/AI engineers (internal team); no non-technical stakeholders for v1
- **Execution environment**: CSV/file-based sandbox for testing phases; PostgreSQL integration in production phase
- **Language**: Python throughout
- **Scoring primary metric**: `non_empty_execution_accuracy` (IBM/unitxt approach — both gold and candidate return data and results match)
- **Guiding principle**: "Fidelity to User Request is Paramount" — intent_fidelity carries highest weight (25%)

## Constraints

- **Tech stack**: Python — no web framework required for v1, CLI only
- **Storage (testing)**: CSV / JSON files — no DB required until Phase 5
- **Storage (production)**: PostgreSQL — migration path is planned, not blocking v1
- **Timeline**: Flexible — quality over speed
- **LLM access**: Required for LLM judge layer; prompt_version must be tracked for reproducibility
- **Reproducibility**: All runs must log model_name + prompt_version to enable regression attribution

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| 4-layer judge pipeline (rule → execution → AST → LLM) | LLM alone is insufficient; deterministic layers catch systematic errors cheaply | — Pending |
| non_empty_execution_accuracy as primary metric | Both queries must return data AND match — empty-result false positives are a known trap | — Pending |
| intent_fidelity at 25% weight | Fidelity to user request is paramount per EvaluateGPT principle | — Pending |
| CSV/JSON first, PostgreSQL later | Reduces setup friction; validation logic is identical regardless of storage | — Pending |
| Judge output quality as separate scoring dimension | Prevents judge hallucination from corrupting eval pipeline silently | — Pending |
| Critical override rules (force reject on parse/exec failure) | Overall score can be misleading when SQL is fundamentally broken | — Pending |

---
*Last updated: 2026-04-01 after initialization*
