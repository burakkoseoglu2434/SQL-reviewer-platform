# Roadmap: SQL Evaluation & LLM-as-a-Judge Platform

## Overview

This platform is built in five phases that follow the natural dependency order of evaluation pipeline construction. Phase 0 locks in the golden dataset schema before any code is written — an irreversible decision that shapes all downstream logic. Phase 1 builds the full deterministic pipeline (rule engine → execution → AST → result comparator), which catches ~70% of failures cheaply and creates the EvalContext architecture everything else depends on. Phase 2 adds the LLM judge, JudgeQualityScorer, and composite scoring engine on top of the stable deterministic foundation. Phase 3 scales the system to batch evaluation with full terminal reporting, hallucination tracking, and judge reliability metrics. Phase 4 completes the observability metadata layer and introduces the ResultStore protocol abstraction that enables the PostgreSQL migration path.

## Phases

**Phase Numbering:**
- Integer phases (0–4): Planned v1.0 work
- Decimal phases (e.g., 2.1): Urgent insertions (marked INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 0: Golden Dataset Foundation** - Enrich, validate, and pre-execute all 968 golden cases before any pipeline code is written
- [ ] **Phase 1: Deterministic Evaluation Pipeline** - End-to-end rule → execution → AST → result-set evaluation of a single case via CLI
- [ ] **Phase 2: LLM Judge + Scoring Engine** - Add LLM judge, JudgeQualityScorer, and composite scoring with 4-state decision output
- [ ] **Phase 3: Batch Runner + Full Reporting** - Replay all golden cases in parallel with terminal report, hallucination tracking, and judge reliability metrics
- [ ] **Phase 4: Observability + Storage Layer** - Capture complete observability metadata per run and introduce the ResultStore protocol abstraction

## Phase Details

### Phase 0: Golden Dataset Foundation
**Goal**: The golden dataset is enriched, validated, and pre-executed so every downstream pipeline layer has a trustworthy baseline from day one
**Depends on**: Nothing (first phase)
**Requirements**: GOLDEN-01, GOLDEN-02, GOLDEN-03, GOLDEN-04, GOLDEN-05
**Success Criteria** (what must be TRUE):
  1. Engineer can run a load command and all 968 rows are loaded with question, gold_sql, gold_answer, and bucket fields present in every row
  2. Every golden case stores execution_metadata (gold_execution_status, gold_row_count, gold_column_names, gold_execution_time_ms, gold_result_fingerprint) — pre-execution confirms all 968 gold SQLs are valid at baseline
  3. Every golden case stores business_context (intent, required_grain, metric_definition, sql_dialect, last_validated_at)
  4. Each case is classified into exactly one bucket (positive/negative/borderline/adversarial) and the CLI reports bucket distribution counts
  5. Engineer can add a new golden case by editing the CSV and re-running the load command; the new case appears in the dataset
**Plans**: TBD

### Phase 1: Deterministic Evaluation Pipeline
**Goal**: A single SQL case can be evaluated end-to-end through all four deterministic layers — rule engine, execution validator, AST validator, result comparator — producing a structured per-layer EvalContext result via one CLI command
**Depends on**: Phase 0
**Requirements**: PIPE-01, PIPE-02, PIPE-03, PIPE-04, RULE-01, RULE-02, RULE-03, RULE-04, EXEC-01, EXEC-02, EXEC-03, EXEC-04, EXEC-05, EXEC-06, AST-01, AST-02, AST-03, AST-04, RESULT-01, RESULT-02, RESULT-03, RESULT-04, RESULT-05
**Success Criteria** (what must be TRUE):
  1. Running `evaluate single_case.json` produces structured output containing rule issues, execution metrics (execution_accuracy, non_empty_execution_accuracy, subset_non_empty_execution_accuracy), AST comparison result, and result-set match ratio — all in one pass
  2. A case with unparseable SQL immediately returns `reject` with parse-failure evidence; no subsequent layers run (early exit confirmed by absence of execution or AST results in output)
  3. DuckDB sandbox executes candidate SQL against CSV virtual tables and reports execution_status, execution_time_ms, result_row_count, and result_columns per case
  4. AST validator produces a normalized sql_ast_hash for both candidate and gold SQL, and marks structurally different but semantically equivalent SQLs as correct without human input
  5. Each pipeline layer passes its unit tests with a mocked EvalContext, confirming no layer calls another layer or accesses storage directly
**Plans**: TBD

### Phase 2: LLM Judge + Scoring Engine
**Goal**: Every evaluated case has a final composite score, 4-state decision, LLM judge rationale with cited evidence, rewrite guidance, and a deterministic judge quality assessment
**Depends on**: Phase 1
**Requirements**: JUDGE-01, JUDGE-02, JUDGE-03, JUDGE-04, JUDGE-05, JUDGE-06, QUAL-01, QUAL-02, QUAL-03, SCORE-01, SCORE-02, SCORE-03, SCORE-04, SCORE-05
**Success Criteria** (what must be TRUE):
  1. LLM judge returns structured JSON with intent_fidelity, business_correctness, and answer_grounding scores (0.0–1.0), rationale, cited evidence, and rewrite_guidance for every evaluated case
  2. Composite score matches the weighted formula (intent_fidelity×0.25 + execution×0.20 + semantic×0.15 + business_correctness×0.15 + result_equivalence×0.10 + answer_grounding×0.10 + risk×0.05) within floating-point tolerance
  3. A case with a known execution error receives `reject` regardless of LLM dimension scores; a case where intent_fidelity<0.20 cannot receive `approve` regardless of composite score
  4. Every judge invocation stores model_name and SHA-256 of the rendered prompt template in the evaluation result — two runs with different prompts produce different hashes
  5. JudgeQualityScorer returns judge_quality_score for every case without making any additional LLM API calls; score is stored as a separate field and excluded from the composite
**Plans**: TBD

### Phase 3: Batch Runner + Full Reporting
**Goal**: All 968 golden cases can be replayed in a single parallel batch run with a full terminal report covering bucket breakdown, hallucination detection, and judge reliability metrics
**Depends on**: Phase 2
**Requirements**: BATCH-01, BATCH-02, BATCH-03, BATCH-04, REPORT-01, REPORT-02, REPORT-03, REPORT-04, REPORT-05, REPORT-06, REPORT-07, REPORT-08, HALL-01, HALL-02, HALL-03, HALL-04, JREL-01, JREL-02, JREL-03, JREL-04
**Success Criteria** (what must be TRUE):
  1. Running `batch-eval` with a golden CSV and candidate directory completes all 968 cases in parallel (asyncio) and prints a rich table report showing pass_rate, non_empty_execution_accuracy, valid_sql_rate, hallucination_rate, avg_latency_ms, and total_token_cost
  2. Batch report breaks down all primary metrics by bucket (positive/negative/borderline/adversarial); a single-bucket filter (`--bucket adversarial`) runs only matching cases
  3. One case failure does not abort the batch; failures are caught and reported as a separate count in the terminal summary alongside the pass rate
  4. Hallucination events (answer hallucination + judge hallucination) are detected per case, logged with type/severity/evidence/case_id, and aggregated as hallucination_rate in the report
  5. Judge reliability block (judge–GT agreement rate, false_positive_rate, false_negative_rate, consistency_score) appears in the batch report when golden cases carry human-assigned ground-truth labels
**Plans**: TBD

### Phase 4: Observability + Storage Layer
**Goal**: Every evaluation run produces a complete, attribution-ready observability record, and the storage layer is abstracted behind a ResultStore protocol that swaps file-backed storage for PostgreSQL with one class change
**Depends on**: Phase 3
**Requirements**: OBS-01, OBS-02, OBS-03, STORE-01, STORE-02, STORE-03
**Success Criteria** (what must be TRUE):
  1. Every judge run writes model_name, latency_ms, input_tokens, output_tokens, prompt_sha256, execution_time_ms, and sql_ast_hash to the evaluation result — two runs with different models produce different model_name entries attributable without manual inspection
  2. Batch run summary includes avg_latency_ms, p95_latency_ms, total_input_tokens, total_output_tokens, and total_token_cost derived from per-case observability metadata
  3. All batch results are written to a JSON file per run via the ResultStore interface; swapping the backend from file to PostgreSQL requires changing one class reference with no pipeline code changes
  4. SQLAlchemy 2 + Alembic schema exists for agent_replay_runs, judge_scores, judge_issues, hallucination_events, and monthly_reports tables — `alembic upgrade head` runs cleanly against a local PostgreSQL instance
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 0 → 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 0. Golden Dataset Foundation | 0/TBD | Not started | - |
| 1. Deterministic Evaluation Pipeline | 0/TBD | Not started | - |
| 2. LLM Judge + Scoring Engine | 0/TBD | Not started | - |
| 3. Batch Runner + Full Reporting | 0/TBD | Not started | - |
| 4. Observability + Storage Layer | 0/TBD | Not started | - |
