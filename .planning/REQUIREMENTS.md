# Requirements: SQL Evaluation & LLM-as-a-Judge Platform

**Defined:** 2026-04-01
**Core Value:** Catch SQL agent errors in evaluation — not in production — by making quality measurable, traceable, and automatically regressionable against a live golden dataset.

## v1 Requirements

### Golden Dataset

- [ ] **GOLDEN-01**: Engineer can load the 968-row CSV into a normalized golden case structure with question, gold_sql, gold_answer, and bucket fields
- [ ] **GOLDEN-02**: Each golden case stores execution_metadata: gold_execution_status, gold_row_count, gold_column_names, gold_execution_time_ms, gold_result_fingerprint
- [ ] **GOLDEN-03**: Each golden case stores business_context: intent, required_grain, metric_definition, sql_dialect, last_validated_at
- [ ] **GOLDEN-04**: Each golden case is classified into a bucket: positive, negative, borderline, or adversarial
- [ ] **GOLDEN-05**: Engineer can add, update, and remove golden cases via CSV edit

### Pipeline Core

- [ ] **PIPE-01**: Engineer can evaluate a single case (candidate_sql + gold_sql + question) through all judge layers with one CLI command
- [ ] **PIPE-02**: Pipeline accumulates all layer outputs in a shared EvalContext object passed sequentially through each layer
- [ ] **PIPE-03**: Pipeline supports early exit: if a critical failure is detected (parse error, execution error) downstream layers are skipped and reject is forced
- [ ] **PIPE-04**: Each pipeline layer is independently unit-testable with a mocked EvalContext

### Rule Engine Layer

- [ ] **RULE-01**: Rule engine checks structural validity by verifying candidate SQL is parseable by sqlglot without errors
- [ ] **RULE-02**: Rule engine detects risk patterns: LIMIT without ORDER BY, CURRENT_DATE/GETDATE() usage, unconstrained Cartesian joins, missing NULL handling
- [ ] **RULE-03**: Rule engine checks business rules: required filter presence, correct table/schema routing, metric definition compliance (configurable per project)
- [ ] **RULE-04**: Rule engine outputs a structured issue list per case: each issue has type, severity (critical/high/medium/low), and evidence string

### Execution Validator Layer

- [ ] **EXEC-01**: Execution validator runs candidate SQL against a DuckDB in-process sandbox that reads CSV/Parquet files as virtual tables
- [ ] **EXEC-02**: Execution validator captures execution_status (success/error/empty), execution_time_ms, result_row_count, result_columns per case
- [ ] **EXEC-03**: Execution validator computes execution_accuracy (SQL runs without error)
- [ ] **EXEC-04**: Execution validator computes non_empty_execution_accuracy: both candidate and gold return data AND their result fingerprints match — this is the primary benchmark metric
- [ ] **EXEC-05**: Execution validator computes subset_non_empty_execution_accuracy: candidate result is a valid subset of gold result

### AST / Semantic Validator Layer

- [ ] **AST-01**: AST validator parses both candidate_sql and gold_sql into ASTs using sqlglot and normalizes them to a canonical dialect form
- [ ] **AST-02**: AST validator computes semantic_equivalence: structurally different SQLs that normalize to the same AST are treated as correct
- [ ] **AST-03**: AST validator stores sql_ast_hash (SHA-256 of normalized AST) per case for deduplication and regression tracking
- [ ] **AST-04**: AST validator fails gracefully when normalization is impossible (dialect edge cases) and defers to result-set comparison as ground truth

### Result Set Comparator Layer

- [ ] **RESULT-01**: Result comparator checks row count match between candidate and gold result sets
- [ ] **RESULT-02**: Result comparator checks column count and column name match (case-insensitive)
- [ ] **RESULT-03**: Result comparator applies numeric tolerance (atol=1e-5) for floating-point value comparison
- [ ] **RESULT-04**: Result comparator performs order-independent comparison by default; strict ordering only when candidate SQL contains explicit ORDER BY
- [ ] **RESULT-05**: Result comparator computes match_ratio for partial match (subset matching) and returns it as a scored dimension

### LLM Judge Layer

- [ ] **JUDGE-01**: LLM judge evaluates intent_fidelity: does the SQL query match the user's literal request including metric, grain, entity, and time scope?
- [ ] **JUDGE-02**: LLM judge evaluates business_correctness: correct table/schema selection, required filters, valid metric definition
- [ ] **JUDGE-03**: LLM judge evaluates answer_grounding: is the natural-language answer fully supported by the SQL result set, with no overclaiming?
- [ ] **JUDGE-04**: LLM judge uses single-item scoring (one case per prompt, no pairwise comparison) to eliminate position bias
- [ ] **JUDGE-05**: LLM judge outputs structured JSON via a Pydantic model containing: per-dimension scores (0.0–1.0), rationale, cited evidence, and rewrite_guidance
- [ ] **JUDGE-06**: Every judge invocation logs model_name and SHA-256 of the rendered prompt template (not just a version label) for reproducibility

### Judge Output Quality Scorer

- [ ] **QUAL-01**: JudgeQualityScorer evaluates judge output using deterministic heuristics only — no second LLM call
- [ ] **QUAL-02**: JudgeQualityScorer measures: specificity (evidence strings non-empty?), completeness (all high-severity issues have rationale?), rationale_density (score-to-reason ratio), actionability (rewrite_guidance non-generic?)
- [ ] **QUAL-03**: Judge quality score is stored as a separate field (judge_quality_score) and not included in the case composite score

### Scoring & Decision

- [ ] **SCORE-01**: Composite score = intent_fidelity×0.25 + execution×0.20 + semantic×0.15 + business_correctness×0.15 + result_equivalence×0.10 + answer_grounding×0.10 + risk×0.05
- [ ] **SCORE-02**: Decision thresholds: ≥0.85→approve, 0.65–0.84→rewrite, 0.40–0.64→human_review, <0.40→reject
- [ ] **SCORE-03**: Critical override rules force reject regardless of overall score: SQL parse failure, SQL execution error, or critical hallucination detected
- [ ] **SCORE-04**: Dimension floor rules: if intent_fidelity<0.20 or business_correctness=0.0 → decision cannot be approve regardless of overall score
- [ ] **SCORE-05**: Structured output per case: decision, overall_score, per-dimension scores, issues list, rewrite_guidance, confidence

### Batch Runner

- [ ] **BATCH-01**: Batch runner replays all golden cases from the dataset and produces a structured result per case
- [ ] **BATCH-02**: Batch runner executes cases in parallel using asyncio with a configurable concurrency limit
- [ ] **BATCH-03**: Batch runner supports filtering by bucket type (run only adversarial cases, only borderline, etc.)
- [ ] **BATCH-04**: One case failure does not abort the batch — failures are caught, logged, and counted separately

### CLI Reporting

- [ ] **REPORT-01**: CLI command `evaluate` accepts a candidate JSON file and golden CSV and prints a single-case evaluation result to terminal
- [ ] **REPORT-02**: CLI command `batch-eval` runs all golden cases and prints a full batch report to terminal using rich tables
- [ ] **REPORT-03**: Batch report includes: pass_rate, non_empty_execution_accuracy, valid_sql_rate, hallucination_rate, avg_latency_ms, total_token_cost, regression_count
- [ ] **REPORT-04**: Batch report breaks down all metrics by bucket (positive/negative/borderline/adversarial)
- [ ] **REPORT-05**: All batch results are exportable to a JSON file for further analysis

### Hallucination Tracking

- [ ] **HALL-01**: System detects answer hallucination: answer claims more scope than the SQL result set supports (e.g., answer says "all products" when SQL has LIMIT 50)
- [ ] **HALL-02**: System detects judge hallucination: judge flags issues that don't exist or approves a case with a critical rule violation
- [ ] **HALL-03**: hallucination_rate is computed per batch run and included in the CLI report
- [ ] **HALL-04**: Each hallucination event is logged with: type (answer/judge), severity, evidence string, and case_id

### Observability

- [ ] **OBS-01**: Every judge run captures: model_name, latency_ms, input_tokens, output_tokens, prompt_sha256, execution_time_ms, sql_ast_hash
- [ ] **OBS-02**: Observability metadata is stored alongside evaluation results to enable regression attribution
- [ ] **OBS-03**: Batch run aggregates: avg_latency_ms, p95_latency_ms, total_input_tokens, total_output_tokens, total_token_cost

### Storage

- [ ] **STORE-01**: All evaluation results are persisted to a JSON file per batch run during the CSV/testing phase (no DB required)
- [ ] **STORE-02**: Storage layer uses a protocol abstraction (ResultStore interface) so the backend swaps from file to PostgreSQL with a single class swap
- [ ] **STORE-03**: SQLAlchemy 2 + Alembic schema is defined for tables: agent_replay_runs, judge_scores, judge_issues, hallucination_events, monthly_reports

## v2 Requirements

### Monthly Governance

- **GOV-01**: Monthly workflow re-executes all gold_sql against the current database, compares result fingerprints, and flags stale cases
- **GOV-02**: Dataset refresh CLI: add new cases, mark deprecated cases, update product_semantics_version
- **GOV-03**: Monthly report includes: score distribution drift, regression list vs previous run, model family distribution change

### Online Evaluation

- **ONLINE-01**: Shadow mode intercepts live agent JSON outputs and evaluates them in real-time against golden dataset context
- **ONLINE-02**: PostgreSQL storage activated for production shadow mode runs

### Advanced Judge

- **ADV-01**: Multi-model judge consensus: run two different LLM families and flag disagreements for human review
- **ADV-02**: Adversarial case auto-generation from failing cases

## Out of Scope

| Feature | Reason |
|---------|--------|
| Web dashboard / UI | CLI output only for v1; web UI adds significant scope |
| Real-time online evaluation | Shadow mode is v2; v1 is batch only |
| Offline model fine-tuning | Evaluation pipeline only — no training loop |
| Multi-tenant access control | Internal single-team tool |
| BIRD benchmark integration | Research reference only; not runtime dependency |
| LangChain / LlamaIndex | Direct API calls preferred; framework adds abstraction overhead |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| GOLDEN-01 through GOLDEN-05 | Phase 0 | Pending |
| PIPE-01 through PIPE-04 | Phase 1 | Pending |
| RULE-01 through RULE-04 | Phase 1 | Pending |
| EXEC-01 through EXEC-05 | Phase 1 | Pending |
| AST-01 through AST-04 | Phase 1 | Pending |
| RESULT-01 through RESULT-05 | Phase 1 | Pending |
| JUDGE-01 through JUDGE-06 | Phase 2 | Pending |
| QUAL-01 through QUAL-03 | Phase 2 | Pending |
| SCORE-01 through SCORE-05 | Phase 2 | Pending |
| BATCH-01 through BATCH-04 | Phase 3 | Pending |
| REPORT-01 through REPORT-05 | Phase 3 | Pending |
| HALL-01 through HALL-04 | Phase 3 | Pending |
| OBS-01 through OBS-03 | Phase 4 | Pending |
| STORE-01 through STORE-03 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 53 total
- Mapped to phases: 53
- Unmapped: 0 ✓

---
*Requirements defined: 2026-04-01*
*Last updated: 2026-04-01 after initial definition*
