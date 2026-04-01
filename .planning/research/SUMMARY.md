# Project Research Summary

**Project:** SQL Evaluation & LLM-as-a-Judge Platform
**Domain:** NL2SQL quality assurance / LLM evaluation pipeline
**Researched:** 2026-04-01
**Confidence:** HIGH (all primary stack choices verified against PyPI, official docs, and academic sources)

---

## Executive Summary

This platform is a **batch evaluation pipeline** for a SQL-generating agent — not a general-purpose LLM evaluation framework. The domain is well-studied (Spider 2.0, BIRD, IBM/unitxt, Alation production systems) and the best-practice architecture has converged: a four-layer deterministic-to-probabilistic funnel where rule checks, SQL execution, and AST comparison filter ~70% of failures cheaply before the LLM judge handles only semantically ambiguous cases. This design keeps evaluation costs low, evaluation speed high, and avoids the core failure mode of LLM-only evaluation (expensive, non-reproducible, subject to systematic judge bias).

The recommended approach is a **Sequential Pipeline with Shared EvalContext**: each evaluator stage is a stateless protocol implementation that writes into a single mutable accumulator, which the pipeline orchestrator converts to an immutable `EvalResult` after all stages complete. The critical architectural insight is that the LLM judge handles only the three dimensions it is uniquely suited for (intent fidelity, business correctness, answer grounding) — execution accuracy, result comparison, and risk checks stay deterministic. The `JudgeQualityScorer` is a deterministic heuristic linter of judge output, not a second LLM call, which prevents infinite regress and keeps cost predictable.

The dominant risks are all on the evaluation validity side, not the implementation side: LLM judge position bias, verbosity bias, and self-enhancement bias corrupt evaluation metrics silently — the pipeline appears to work but produces scores that don't reflect real quality. These must be designed out in Phase 1 before any evaluation data is trusted. The second major risk is golden dataset label rot (BIRD benchmark had 52.8% annotation error rate per CIDR 2026) — the gold SQL must be re-executed on every run and freshness-tracked from day one.

---

## Key Findings

### Recommended Stack

The stack is unusually well-determined for this domain — every primary choice has a clear technical rationale and verified production usage. The only judgment call is whether to use `instructor` for multi-provider LLM abstraction (MEDIUM confidence, optional).

**Core technologies:**

| Library | Version | Purpose | One-line rationale |
|---------|---------|---------|-------------------|
| `sqlglot[c]` | `>=25.0,<31` | SQL parsing, AST traversal, dialect normalization | True parser (not tokenizer); supports `diff()` for structural comparison, `optimizer.normalize` for canonical form; 45.8M monthly PyPI downloads |
| `duckdb` | `>=1.5.0` | CSV/file-based SQL execution sandbox | In-process, zero-config, reads CSV as virtual tables, PostgreSQL-compatible SQL dialect — swap to real PG with one protocol class change |
| `openai` / `anthropic` | `>=1.60.0` / `>=0.50.0` | LLM judge API clients | Native structured output via `client.beta.chat.completions.parse()` with Pydantic models — no JSON schema boilerplate, SDK handles retries |
| `pydantic` v2 + `pydantic-settings` | `>=2.8` / `>=2.3` | Pipeline data models, structured judge output, typed config | `BaseModel` for all I/O contracts; `BaseSettings` for `.env` config; required for native SDK structured output |
| `pandas` + `numpy` | `>=2.2` / `>=1.26` | Result set comparison, numeric tolerance | `pd.testing.assert_frame_equal(check_exact=False, rtol=1e-3)` is the standard pattern; DuckDB returns pandas DataFrames natively via `fetchdf()` |
| `typer` + `rich` | `>=0.12` / `>=14.0` | CLI runner, terminal output | Type-hint-based CLI auto-generates `--help`; Rich renders pass-rate tables, progress bars, and regression diffs with zero effort |
| `pytest` + `pytest-mock` | `>=8.0` / `>=3.14` | Testing, LLM mock fixtures | `@pytest.mark.parametrize` for golden dataset sampling; `mocker.patch()` prevents live API calls in CI |

**Phase 2 additions (PostgreSQL migration):** `sqlalchemy>=2.0`, `alembic>=1.13`, `psycopg2-binary>=2.9`

**Optional:** `instructor>=1.4.0` — unified structured output wrapper across providers; use if you need to swap judge model providers without code changes.

**What NOT to use:** sqlparse (tokenizer, not parser), LangChain/LlamaIndex (15+ transitive deps for a use case native SDKs handle natively), SQLite (OLTP row-store; slower for analytical eval queries).

---

### Table Stakes Features

Features any credible SQL evaluation platform must have. Missing any = platform is unusable or produces misleading metrics.

| Feature | Why It's Non-Negotiable |
|---------|------------------------|
| Execution accuracy (run both SQLs, compare result sets) | The foundational correctness signal |
| Order-independent, column-name-insensitive comparison | SQL result order is not semantically meaningful; LLMs vary column alias naming |
| Numeric tolerance (`atol=1e-5`) | Floating-point imprecision causes false negatives on valid SQL — IBM/unitxt documented standard |
| SQL parseability check (hard fail before any other check) | Unparseable SQL must be caught and short-circuited before execution |
| Batch runner (replay full golden dataset) | Running 1 case manually produces no signal; the platform only works at batch scale |
| Golden dataset ingestion from CSV/JSON | Schema: question, gold_sql, gold_answer, bucket, business_context |
| Per-case structured output (score + decision + issues) | "Failed" is not actionable; engineers need to know *why* |
| Aggregate batch report (pass_rate, failure breakdown) | The primary quality dashboard signal after every run |
| LLM judge for semantic equivalence + intent | Execution accuracy produces false negatives (different SQL, correct answer) and false positives (empty-result matching) |
| Error classification (syntax / semantic / business_logic / hallucination) | Unclassified failure is unactionable |
| Model + prompt version logging per run | Without version tracking, regressions cannot be attributed to model vs. prompt changes |

---

### Differentiators

What elevates this platform beyond basic execution accuracy checking. These are what make evaluation genuinely diagnostic rather than just a pass/fail gate.

**High-value, implement in v1:**

- **Non-empty execution accuracy (IBM/unitxt)** — both gold and candidate must return data AND match; eliminates empty-result false-positive trap that is well-documented in production systems
- **4-layer pipeline (rule → execution → AST → LLM)** — LLM-only eval is expensive and slow; deterministic layers catch ~70% of failures in <0.5s; LLM runs only on ambiguous cases (10-20% of batch)
- **4-state decision output** (approve / rewrite / human_review / reject) — binary pass/fail throws away gradient signal; "rewrite" and "human_review" guide the agent's next action
- **Critical override rules** — composite score of 0.65 with an unparseable SQL should never suggest "rewrite"; overrides enforce correctness floors regardless of composite
- **Bucket-aware evaluation** (positive / negative / borderline / adversarial) — aggregate pass rate hides bucket-level regressions; an agent may improve on easy cases while regressing on adversarial ones
- **Observability metadata per run** (latency_ms, input/output tokens, prompt_version, sql_ast_hash) — performance regressions matter as much as quality regressions; token tracking enables cost attribution
- **Rewrite guidance in judge output** — knowing SQL failed is less valuable than knowing *how* to fix it

**Implement after core pipeline is stable:**

- **AST/semantic validator layer** — catches semantic divergence without execution; complex to tune correctly (see Pitfall C-5); execution + LLM covers most cases first
- **Judge Output Quality scorer** — meta-evaluates judge specificity, completeness, rationale density, actionability; build after judge output is stable
- **Regression list** (cases that passed before, fail now) — requires two stored runs; implement after storage layer is in place
- **Multi-dimension scoring (8 dimensions)** with per-dimension breakdown — full diagnostic profile; enables directing fixes to the right layer

**Defer to v2+:**
- Web dashboard / UI (CLI is sufficient for internal ML team)
- Real-time / shadow mode evaluation (fundamentally different architecture)
- Multi-dialect SQL support (PostgreSQL-first; dialect migration is a config change once architecture is proven)
- Interactive prompt playground (use Jupyter notebook for prompt iteration; evaluation pipeline consumes prompt versions)

---

### Architecture Pattern

The canonical pattern is a **Sequential Pipeline with Shared EvalContext** — not pure event-driven, not chain-of-responsibility. Each layer receives an immutable input context and enriches a single mutable `EvalContext` accumulator. Layers are ordered by computational cost (cheapest first) and short-circuit on critical failures via a `critical_failure` flag checked by the pipeline orchestrator at each stage boundary.

**5 structural pillars:**

1. **Forward-only data flow** — no evaluator calls another evaluator; no evaluator writes to storage; all cross-stage communication goes through `EvalContext`; `ResultStore` is the only component with persistence responsibility
2. **Protocol-based abstraction at every I/O boundary** — `SQLExecutor` protocol enables CSV sandbox in dev and PostgreSQL in production with zero pipeline changes; `ResultStore` protocol enables JSONL in v1 and PostgreSQL in v2; `CaseRepository` protocol abstracts golden dataset storage
3. **Pipeline orchestrator owns all routing logic** — individual evaluators set `context.critical_failure = True` but never decide whether to stop; the orchestrator checks the flag at each stage boundary; this keeps routing in one place and evaluators purely functional
4. **JudgeQualityScorer is deterministic, not LLM-based** — scores judge output using heuristic rules (evidence specificity via regex, reasoning word count, rewrite guidance structure); avoids infinite regress and keeps cost predictable
5. **Versioned prompt registry** — every LLM judge call logs `prompt_version` + SHA-256 of rendered prompt; regression detection compares `(model_name, prompt_version)` pairs; prompt changes require a quarantine calibration run before official use

**Build order (wave dependencies):**
```
Wave 0 → Core dataclasses + protocols (no deps)
Wave 1 → Storage backends + dataset loading (deps: Wave 0)
Wave 2 → Deterministic evaluators: Normalizer, RuleEngine, ASTValidator, ResultComparator (deps: Wave 0+1)
Wave 3 → ExecutionValidator via DuckDB (deps: Wave 1+2)
Wave 4 → LLMJudge + JudgeQualityScorer (deps: Wave 0+2+3)
Wave 5 → ScoreAggregator + EvaluationPipeline (deps: Wave 2+3+4)
Wave 6 → BatchRunner + RegressionTracker + ReportGenerator (deps: Wave 1+5)
Wave 7 → PostgreSQL migration (deps: all prior)
```

---

### Critical Pitfalls

The top 5 project-killers, with one-line prevention each. All 5 corrupt evaluation metrics silently — the pipeline appears to work while producing scores that don't reflect reality.

| # | Pitfall | Impact | Prevention (one line) |
|---|---------|--------|----------------------|
| C-1 | **LLM Judge Position Bias** — LLM systematically favors the first-presented SQL regardless of quality (confirmed across 15 judges, 150K instances) | Scores are non-reproducible; regression detection breaks | Use single-item scoring (judge candidate independently without gold SQL in same prompt) or run dual ordering and average |
| C-2 | **Verbosity Bias** — judge assigns higher scores to longer, more elaborate SQL independent of correctness (RLHF-baked) | Verbose-but-wrong SQL outscores terse-but-correct; `judge_quality_score` inflated for padding | Add explicit anti-verbosity instruction: *"Longer is not better. Score only on correctness, not on length."* |
| C-3 | **Judge Self-Enhancement / Family Bias** — same-family judge and agent share failure modes; judge cannot penalize what it cannot detect (AWS research confirmed for GPT-4o and Claude 3.5 Sonnet) | Evaluation metrics inflated; true regressions underreported | Use a different model family for judge than for the SQL agent being evaluated |
| C-4 | **Composite Score Masking Critical Failures** (Goodhart's Law) — a 0.0 on a 15%-weighted dimension reduces composite by only 0.15, not enough to cross threshold | Catastrophic semantic errors receive "approve" decisions | Add dimension floor rules: `intent_fidelity < 0.4` → force human_review; `business_correctness < 0.3` → force human_review; extend beyond execution-error overrides |
| C-5 | **AST Normalization False Equivalence** — sqlglot can over-normalize (different queries marked equivalent) and under-normalize (same-result queries marked different); known failure modes for CTE vs. subquery, table aliases, `LEFT ANTI JOIN` | AST semantic layer becomes noisy; developers lose trust; distorts composite score | Use result-set equivalence as ground truth; AST comparison is a fast pre-filter that produces `needs_verification`, never final `reject`; define canonical form for your dialect before implementing |

**Moderate pitfalls to address in Phase 1:**
- **M-1 (SQL Sandbox Injection)**: SELECT-only allowlist + timeout enforcement + read-only DuckDB connection
- **M-2 (Golden Dataset Label Rot)**: Re-execute gold SQL on every run; track `last_validated_at`; flag stale fingerprints before publishing metrics
- **M-4 (Prompt Version Drift)**: Hash the full rendered prompt (SHA-256), not just the version label; pin dated model snapshots (`gpt-4o-2024-11-20`) not floating aliases

---

## Implications for Roadmap

### Phase 0: Golden Dataset Foundation
**Rationale:** Everything downstream depends on the golden dataset schema being correct. Adding `dialect`, `expected_tables`, `business_context`, `last_validated_at`, and `gold_result_fingerprint` fields before any code is written prevents re-ingestion later. Label rot (Pitfall M-2) starts here.
**Delivers:** Enriched golden dataset schema; validated 968-row CSV with bucket classification; `gold_sql` pre-executed to confirm all gold SQL is valid at baseline
**Addresses:** Table stakes features — golden dataset ingestion, schema validation
**Avoids:** M-2 (Label Rot), C-5 (AST dialect annotation missing), M-5 (schema mismatch blindness from missing `expected_tables`)
**Open questions:** What buckets exist in current CSV? Does `business_context` need manual annotation or can it be derived? Are all 968 gold SQLs currently executable?

### Phase 1: Core Evaluation Pipeline (Wave 0–5)
**Rationale:** Implement the full pipeline in build-order dependency sequence: data models → storage backends → deterministic evaluators → execution validator → LLM judge → aggregator. This is the critical path. Everything else (reporting, regression tracking, PostgreSQL) depends on this being correct.
**Delivers:** End-to-end evaluation of a single case: rule check → execution → AST comparison → LLM judge → composite score → 4-state decision
**Uses:** sqlglot (AST), duckdb (CSV sandbox), openai/anthropic + pydantic (LLM judge), pydantic-settings (config)
**Implements:** EvalContext pattern, BaseEvaluator protocol, SQLExecutor protocol, PromptRegistry, JudgeQualityScorer (deterministic heuristics)
**Avoids:** C-1 (dual-ordering or single-item scoring baked into first judge prompt), C-2 (anti-verbosity instruction), C-3 (cross-family judge selection), C-4 (dimension floor rules in ScoreAggregator), M-1 (SELECT-only sandbox)
**Open questions:** Which model family is the SQL agent using? (determines judge model selection for C-3). Zero-shot or few-shot judge? (few-shot requires deliberate calibration to avoid M-3 anchoring bias). What is the first `prompt_version` naming scheme?
**Research flag:** LLM judge prompt engineering needs dedicated investigation — positional bias mitigation patterns, rubric structure, calibration anchor selection.

### Phase 2: Batch Runner + CLI Reporting (Wave 6)
**Rationale:** Single-case evaluation is not useful without batch replay. Reporting must include per-bucket breakdown from day one — aggregate metrics hide bucket-level failures (Pitfall m-5).
**Delivers:** Full batch replay against all 968 golden cases; per-case JSONL output; aggregate CLI report with pass_rate, non_empty_execution_accuracy, hallucination_rate, per-bucket breakdown, avg_latency, token_usage
**Uses:** typer + rich (CLI), asyncio + semaphore (concurrency — reduces 30min sequential LLM run to ~3min at 10 concurrent)
**Avoids:** m-3 (deterministic gating before LLM judge — only invoke judge on cases that pass rule + execution layers), m-5 (mandatory per-bucket breakdown)
**Open questions:** What concurrency limit works within API rate limits? What is the baseline pass_rate target for the first production run?

### Phase 3: Observability + Regression Tracking
**Rationale:** Regression detection requires at minimum two stored runs with consistent schema. Prompt drift detection requires SHA-256 hash of rendered prompt per run. Without this, monthly evaluations cannot be trusted as trend indicators.
**Delivers:** Regression list (cases that passed before, fail now); score distribution monitoring (mean/std/skew per dimension per run — alerts on >0.1 std drift); prompt hash stored in every run; `judge_model_name` tracked separately from `agent_model_name`
**Avoids:** M-4 (prompt version drift attribution errors), M-3 (anchoring bias — score distribution alert), C-3 (cross-judge calibration workflow)
**Open questions:** Where is the baseline run stored? What is the score distribution alert threshold? What format for the regression list — in-terminal or written to file?

### Phase 4: Monthly Governance Workflow
**Rationale:** Without a defined process, golden dataset labels rot (Pitfall M-2) and adversarial cases become trivial over time (Pitfall m-4). This phase makes evaluation sustainable, not just functional.
**Delivers:** Automated gold SQL re-execution with stale fingerprint flagging; adversarial case freshness protocol (refresh trigger at 80% pass rate per failure mode category); monthly report publication checklist; human reviewer assignment workflow for stale cases
**Avoids:** M-2 (Label Rot), m-4 (Adversarial decay)
**Research flag:** Governance tooling is project-specific; no standard library patterns apply — implement as lightweight CLI subcommands.

### Phase 5: PostgreSQL Storage Migration (Wave 7)
**Rationale:** CSV/JSONL storage hits limits once regression tracking requires querying across multiple runs. PostgreSQL with SQLAlchemy 2 is the pre-planned migration target; protocol abstractions in Phase 1 make this a class-swap, not a rewrite.
**Delivers:** PostgreSQL-backed `agent_replay_runs`, `judge_scores`, `judge_issues`, `hallucination_events`, `monthly_reports` tables; SQLAlchemy 2 ORM models; Alembic migrations; production `PostgreSQLExecutor` replacing CSV sandbox
**Uses:** sqlalchemy>=2.0, alembic>=1.13, psycopg2-binary>=2.9
**Open questions:** Is PostgreSQL already running in the target environment? What is the target schema for time-series regression queries?

---

### Phase Ordering Rationale

- **Phase 0 before all code**: Dataset schema decisions are irreversible once the pipeline is built around them; fixing the schema retroactively requires re-ingestion of all 968 cases
- **Critical path is Wave 0→1→3→4→5→6**: Data models must precede storage, which must precede execution, which must precede LLM, which must precede aggregation, which must precede batch running
- **Pitfalls C-1 through C-4 must be addressed in Phase 1**: Judge bias issues cannot be corrected retroactively once evaluation data has been collected; the first run with position bias produces historically corrupted baselines
- **PostgreSQL deferred to Phase 5**: Same evaluation logic applies regardless of storage backend; CSV/JSONL is sufficient for 968 cases and avoids setup friction while the pipeline is being validated

### Research Flags

**Needs deeper research during planning:**
- **Phase 1 (LLM Judge)**: Positional bias mitigation patterns; calibration anchor selection; zero-shot vs. few-shot tradeoffs; rubric structure for multi-dimension scoring. Academic literature is dense and findings are non-obvious.
- **Phase 1 (AST Validator)**: Canonical SQL form definition for PostgreSQL/DuckDB dialect; sqlglot normalization edge cases for specific query patterns in the golden dataset.

**Standard patterns — skip research-phase:**
- **Phase 0 (Dataset)**: CSV enrichment; no novel patterns needed
- **Phase 2 (Batch Runner)**: asyncio semaphore pattern is well-documented
- **Phase 3 (Observability)**: SHA-256 hashing, JSONL logging — standard Python
- **Phase 5 (PostgreSQL)**: SQLAlchemy 2 + Alembic migration is well-documented; `from_attributes=True` for Pydantic v2 integration is confirmed

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All primary choices verified against PyPI, official release notes, and official docs; no speculative picks |
| Features | HIGH | Based on production systems (Alation, IBM/unitxt), academic benchmarks (Spider 2.0, BIRD), and published evaluation platforms (GRAFITE, EvalAssist) |
| Architecture | HIGH | Sequential pipeline pattern is well-established; key interface contracts (BaseEvaluator, SQLExecutor, ResultStore protocols) are directly derived from project requirements + verified DuckDB/Pydantic patterns |
| Pitfalls | HIGH | All critical pitfalls are backed by peer-reviewed papers (ACL 2025, CIDR 2026) or AWS production research with quantified impact |

**Overall confidence:** HIGH

### Gaps to Address During Planning

- **Business context annotation**: The `business_context` field in the golden dataset schema is required for the `business_correctness` dimension — it is currently undefined. Before Phase 1, decide whether this is manually annotated, derived from question metadata, or approximated from domain context.
- **Adversarial case tags**: Existing 968-row dataset has no `adversarial_target` annotations. Need to decide: tag during Phase 0, or defer adversarial tagging to Phase 4?
- **Judge model selection**: Cannot finalize until the SQL agent's model family is confirmed (Pitfall C-3 requires cross-family selection). Placeholder: if agent uses GPT-4o family, use Claude Sonnet as judge and vice versa.
- **Score distribution baselines**: The first run has no historical baseline for distribution monitoring. Plan: run Phase 1 on a 100-case calibration subset first; establish distribution baseline before running the full 968-case batch.

---

## Sources

### Primary (HIGH confidence)
- sqlglot: https://pypi.org/project/sqlglot/ (v30.1.0, March 2026)
- duckdb: https://duckdb.org/2026/03/23/announcing-duckdb-151.html
- OpenAI structured outputs: https://developers.openai.com/docs/guides/structured-outputs
- Anthropic structured outputs: https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs
- pydantic-settings: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- LLM-as-a-Judge (pydantic evals): https://pydantic.dev/articles/llm-as-a-judge
- IBM/unitxt `SQLExecutionAccuracy` + `non_empty_execution_accuracy`: https://www.unitxt.ai/

### Secondary (MEDIUM confidence)
- Alation Engineering Blog — LLM-as-a-Judge for SQL (Sep 2025) — execution accuracy evolution, heuristic variants
- dev.to production SQL eval guide — two-layer architecture patterns
- GRAFITE (2025) — regression tracking platform patterns
- EvalAssist IBM/AAAI 2025 — trustworthiness mechanisms

### Tertiary / Academic (HIGH confidence for pitfall findings)
- Zheng et al. (2025). "Judging the Judges: Systematic Study of Position Bias." ACL IJCNLP 2025. https://aclanthology.org/2025.ijcnlp-long.18/
- CalibraEval (2025). "Calibrating Prediction Distribution to Mitigate Selection Bias." ACL 2025. https://aclanthology.org/2025.acl-long.808/
- BIRD Annotation Error Rate (2026). "Text-to-SQL Benchmarks Are Broken." CIDR 2026. https://vldb.org/cidrdb/2026/text-to-sql-benchmarks-are-broken-an-in-depth-analysis-of-annotation-errors.html
- ToxicSQL (2025). "Are Your LLM-based Text-to-SQL Models Secure?" arXiv:2503.05445
- Goodhart's Law in Agent Evaluation (2026). https://www.agentpact.ai/labs/research/2026-03-17-goodharts-law-agent-evaluation-gaming
- Anchoring Bias in LLMs (2025). arXiv:2511.05766

---
*Research completed: 2026-04-01*
*Ready for roadmap: yes*
