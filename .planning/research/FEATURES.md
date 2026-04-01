# Feature Landscape: SQL Evaluation & LLM-as-a-Judge Platform

**Domain:** SQL evaluation / NL2SQL quality assurance / LLM-as-a-judge
**Researched:** 2026-04-01
**Sources:** Alation Engineering Blog, IBM/unitxt, dev.to production SQL eval guide, Spider 2.0 / BIRD benchmarks, Arize Phoenix, MLflow, LangSmith, GRAFITE regression platform, EvalAssist (IBM/AAAI 2025)

---

## Table Stakes

Features users/engineers expect from any SQL evaluation platform. Missing any of these = platform is unusable or misleading.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Execution accuracy** — run candidate SQL, compare result set to gold | The foundational metric; binary correctness signal | Low | Simple row/col match on result sets |
| **Result set comparison: order-independent** | SQL result order is not semantically meaningful unless ORDER BY is explicit | Low | Sort both sets before comparing |
| **Result set comparison: column-insensitive names** | LLMs vary column alias naming; same data with different aliases is still correct | Low | Case-fold + alias-strip before compare |
| **Numeric tolerance** | Floating-point imprecision causes false negatives on valid SQL | Low | `atol=1e-5` tolerance is standard (unitxt approach) |
| **SQL parseability check** | Unparseable SQL is a hard failure; must be caught before any other check | Low | sqlglot or equivalent parser |
| **Batch evaluation runner** | Running 1 case manually is useless; must replay full golden dataset | Medium | Async + concurrency control |
| **Golden dataset ingestion** — load from CSV/JSON | Without a reference dataset, nothing can be evaluated | Low | Schema: question, gold_sql, gold_answer, bucket |
| **Per-case structured output** — score + decision + issues | Engineers must know why each case passed or failed, not just aggregate pass rate | Medium | JSON output per case |
| **Aggregate batch report** — pass_rate, accuracy, failure breakdown | Overall quality signal; the thing you look at after every run | Low | CLI table or JSON summary |
| **LLM judge layer** — semantic equivalence, intent fidelity | Execution accuracy produces false negatives (correct answer, different SQL structure) and false positives (empty-result matching); LLM resolves ambiguity | High | Requires prompt engineering + structured JSON output |
| **Error classification** — why did this case fail | "Failed" is not actionable; "missing JOIN on customer_id" is | Medium | Categories: syntax, semantic, business_logic, hallucination |
| **Structured judge output** — JSON with score, issues, reasoning | Free-text judge output cannot be aggregated or trended | Medium | Forces LLM to output parseable JSON |
| **Pass/fail decision per case** | Engineers need a binary gate signal to catch regressions at a glance | Low | Derived from composite score threshold |
| **Reproducibility: model + prompt version logging** | Without version tracking, you cannot attribute regressions to prompt changes vs model changes | Low | Metadata field per run: `model_name`, `prompt_version` |

---

## Differentiators

Features that elevate this platform beyond basic execution accuracy checking. These are what make the platform exceptionally useful for a production SQL agent team.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Non-empty execution accuracy (IBM/unitxt approach)** | Standard execution accuracy produces false positives when both gold and candidate SQL return empty results — looks like a match but isn't informative. Non-empty variant requires BOTH to return data AND match. | Low | Primary metric; prevents a common false-positive trap well documented in unitxt |
| **4-layer pipeline (rule → execution → AST → LLM)** | LLM-only evaluation is expensive and slow. Deterministic layers filter obvious failures in <0.5s so LLM only runs on ambiguous cases. 80/20: rules catch ~70% of failures cheaply. | High | Requires implementing all 4 layers as independent, composable modules |
| **AST/semantic validator layer** | Two SQLs can produce the same result on the test DB but differ semantically (SPOTIT research finding). AST normalization + structural comparison catches semantic divergence without execution. | High | Uses sqlglot; normalize aliases, whitespace, column ordering before compare |
| **Multi-dimension scoring (8 dimensions)** | Single execution accuracy misses intent drift (user asked for Q1 but got full year), business logic errors (wrong KPI formula), and answer grounding failures. 8 dimensions give a full diagnostic profile. | High | intent_fidelity 25%, execution 20%, semantic 15%, business_correctness 15%, result_equivalence 10%, answer_grounding 10%, risk 5% |
| **Critical override rules** (force reject on parse/exec failure) | Composite scores can be misleading when SQL is fundamentally broken — 0.65 composite with an unparseable SQL would incorrectly suggest "rewrite" rather than "reject." Overrides enforce correctness floors. | Low | Override conditions: execution error, unparseable SQL, critical hallucination |
| **4-state decision output** (approve / rewrite / human_review / reject) | Binary pass/fail throws away gradient signal. "Rewrite" and "human_review" states guide the agent's next action intelligently rather than defaulting everything below threshold to reject. | Low | Thresholds: ≥0.85 approve, 0.65–0.84 rewrite, 0.40–0.64 human_review, <0.40 reject |
| **Rewrite guidance in judge output** | Knowing a SQL failed is less valuable than knowing *how* to fix it. Rewrite guidance enables closed-loop improvement without human intervention. | Medium | LLM judge generates a corrected SQL suggestion when verdict is FAIL/PARTIAL |
| **Judge output quality scoring (meta-evaluation)** | Judge hallucination is a silent pipeline corruption risk — if the judge fabricates issues, every downstream metric is wrong. Meta-scoring the judge's own output quality prevents this. | Medium | Dimensions: specificity, completeness, rationale_density, actionability |
| **Hallucination tracking — both answer and judge** | Answer hallucination (SQL returns data that doesn't exist) and judge hallucination (LLM fabricates issues that aren't present) are distinct failure modes with different mitigations. Tracking both separately provides actionable telemetry. | Medium | Separate hallucination_events log per case, per type |
| **Bucket-aware evaluation** (positive / negative / borderline / adversarial) | Aggregate pass rate hides bucket-level degradation. An agent may improve on easy cases while regressing on adversarial ones. Bucket breakdown reveals the real quality gradient. | Low | Four buckets in golden dataset; report pass rate per bucket |
| **Risk/reliability dimension** (LIMIT, CURRENT_DATE, NULL, JOIN multiply patterns) | SQL agents frequently produce queries that are syntactically valid and semantically close but operationally dangerous (missing LIMIT on large tables, using CURRENT_DATE in a non-deterministic way). A dedicated risk dimension flags these before production. | Medium | Rule engine checks for known risk patterns |
| **Observability metadata per run** (latency_ms, input/output tokens, execution_time_ms, sql_ast_hash) | Performance regressions (slower, more expensive) matter as much as quality regressions. Token tracking enables cost attribution to specific judge model versions. | Low | Logged as structured metadata alongside evaluation results |
| **SQL AST hash for semantic deduplication** | Same logical SQL written differently will have different text hashes. AST hash enables detecting when two SQL variants are semantically identical, preventing false regression alerts from cosmetic changes. | Low | Hash the normalized AST, not the raw SQL string |
| **Regression list in batch report** — cases that passed before but fail now | A pass rate improvement can hide individual case regressions. Explicitly surfacing regressions prevents "two steps forward, one step back" silent degradation. | Medium | Requires comparing current run to previous baseline run |
| **Per-dimension score breakdown in output** | When a case fails, knowing which dimension failed (execution vs intent vs business_correctness) directs the fix to the right layer, rather than treating all failures as equivalent. | Low | Already part of structured output schema |
| **Answer grounding dimension** | Checks that the agent's natural-language answer is supported by the actual result data — catches cases where SQL is correct but the answer narrative hallucinates or misrepresents the data. | Medium | LLM judge evaluates answer-to-result coherence |
| **Business correctness dimension** | Technical SQL correctness (valid, executes, matches rows) does not guarantee business correctness (right KPI definition, right date range, right aggregation level). This dimension flags business logic errors that execution accuracy misses entirely. | High | Requires business context metadata in golden dataset |

---

## Anti-Features

Features to deliberately NOT build in v1. Each deferred item has a clear rationale.

| Anti-Feature | Why Avoid in v1 | What to Do Instead |
|--------------|-----------------|-------------------|
| **Web dashboard / UI** | Adds framework dependency (Flask, FastAPI, frontend build), auth requirements, and 2-4 weeks of non-evaluation work. CLI output is sufficient for an internal ML team. | Rich terminal output (tables, color, summary) via `rich` or `tabulate`. Upgrade path: export JSON → connect any BI tool later. |
| **Real-time / online / shadow mode evaluation** | Requires integration with the live inference pipeline, event streaming, and async workers. Fundamentally different architecture from batch evaluation. | Batch replay against golden dataset covers the same correctness signal with zero infrastructure complexity. Plan for Phase 5. |
| **Offline fine-tuning / model retraining** | Evaluation pipeline and training pipeline are architecturally separate. Mixing them creates coupling that makes evaluation results unreliable (the model being evaluated is also being changed). | Evaluation outputs (rewrite guidance, issue classifications) can feed a retraining pipeline as a downstream consumer — but that pipeline is separate. |
| **Multi-tenant access control / OAuth** | Internal tool with a single team. RBAC adds authentication infrastructure, session management, and permission logic that provides zero value for the first year. | File-based access. If needed later, wrap CLI behind a simple API with API key auth. |
| **Public leaderboard / benchmark sharing** | This is a custom golden dataset tied to a specific business domain. Publishing it creates IP exposure and forces maintaining a public-facing benchmark with versioning semantics. | Keep dataset internal. If academic contribution is desired, create a sanitized public version as a separate initiative. |
| **Multi-dialect SQL support** (PostgreSQL + MySQL + BigQuery + Snowflake) | Each SQL dialect requires its own execution sandbox, parser configuration, and result normalization rules. Premature generalization before the core pipeline is stable adds 3-4x scope. | Pin to PostgreSQL. Use `sqlglot` dialect parameter — migration to other dialects is a config change once architecture is proven. |
| **Interactive prompt playground** | Useful for prompt engineering exploration but not for systematic evaluation. Adds UI complexity and shifts focus from batch correctness to single-case tinkering. | Use a separate Jupyter notebook or LiteLLM playground for prompt iteration. Evaluation pipeline consumes prompt versions, not designs them. |
| **Automatic model comparison / A/B testing infrastructure** | Model comparison is a separate workflow (run eval twice with different model configs and diff results). Building it as a platform feature prematurely creates a product scope problem before the core pipeline is validated. | Run eval with `model_name=A`, then with `model_name=B`, compare the output JSON files. The prompt_version + model_name metadata fields are designed to support this manually first. |
| **Streaming / incremental evaluation** | Streaming evaluation introduces partial state management, timeout handling, and result consistency problems. Not needed when batch replay is already fast enough for 968 cases. | Async batch with concurrency control (semaphore) covers throughput needs. |
| **Natural language query interface to evaluation results** | Asking "which cases failed on business_correctness?" via chat is a nice feature but adds LLM overhead on top of the evaluation infrastructure itself. | Use structured JSON output + standard CLI filtering (`jq`, `grep`, Python scripts). Upgrade path: structured query via pandas after PostgreSQL integration. |

---

## Feature Dependencies

```
Golden dataset schema (question, gold_sql, gold_answer, bucket, business_context)
    └── Rule Engine Layer (structural + risk checks)
    └── Execution Validator Layer
            └── non_empty_execution_accuracy (both must return data + match)
            └── Result Set Comparator (row count, col names, numeric tolerance)
    └── AST/Semantic Validator Layer
            └── SQL AST hash (deduplication, regression detection)
    └── LLM Judge Layer
            └── intent_fidelity, business_correctness, answer_grounding
            └── Rewrite guidance
            └── Judge Output Quality scorer (meta-evaluation)
                    └── Hallucination tracking (judge type)
    └── Composite Scoring
            └── Critical override rules
            └── 4-state decision (approve/rewrite/human_review/reject)
    └── Per-case structured output
            └── Batch runner
                    └── Aggregate CLI report
                            └── Regression list (requires baseline comparison)
                            └── Bucket-aware breakdown
                            └── Observability metadata (latency, tokens)
```

**Key dependency chains:**

- **Regression list** requires at minimum two runs stored with the same schema → depends on CSV/JSON persistence being consistent across runs
- **Judge output quality scoring** depends on the LLM judge producing structured JSON → judge must output parseable JSON before meta-scoring can run
- **Non-empty execution accuracy** depends on the execution sandbox being able to run both gold and candidate SQL against the same data state
- **Business correctness** depends on `business_context` field being populated in the golden dataset → must be addressed in golden dataset enrichment phase before this dimension is valid
- **Hallucination tracking (judge)** depends on judge output quality scorer being implemented first → the quality scorer surfaces the evidence; hallucination event is logged when specificity/rationale_density fall below threshold

---

## MVP Recommendation

**Prioritize (v1):**
1. Golden dataset ingestion + schema (foundation for everything)
2. Execution validator + result set comparator (fastest correctness signal)
3. Rule engine (cheap, catches ~40% of failures deterministically)
4. Non-empty execution accuracy as primary metric (avoids known false-positive trap)
5. LLM judge with structured JSON output (handles semantic ambiguity)
6. Composite scoring + critical overrides + 4-state decision
7. CLI batch report with per-case output + aggregate summary
8. Observability metadata per run (essential for regression attribution)

**Defer to later phases:**
- AST/semantic validator (adds coverage but complex; execution + LLM covers most cases first)
- Judge output quality scorer (meta-evaluation; build after judge is stable)
- Regression list (requires two stored runs; defer until storage is in place)
- PostgreSQL migration (start with CSV/JSON, same logic applies)
- Real-time/online evaluation (Phase 5 per PROJECT.md)

---

## Sources

- Alation Engineering Blog: LLM-as-a-Judge for SQL (Sep 2025) — execution accuracy evolution, heuristic variants, LLM judge alignment methodology
- dev.to: "Build a Production-Ready SQL Evaluation Engine for LLMs" — two-layer architecture, deterministic + AI judge, async batch patterns
- IBM/unitxt docs (v1.26.6): `SQLExecutionAccuracy`, `non_empty_execution_accuracy` — metric definitions
- IBM/text2sql-eval-toolkit (GitHub) — execution-based metrics, semantic equivalence challenges
- Spider 2.0 (ICLR 2025 Oral) — enterprise benchmark; only 21.3% task success with o1-preview; enterprise complexity vs Spider 1.0
- BIRD Benchmark analysis — large-scale database evaluation, domain-specific schema complexity
- SPOTIT (2025 paper) — formal verification shows test-based methods miss semantic differences between SQL queries
- GRAFITE (2025) — regression tracking platform; issue repository, side-by-side model comparison
- EvalAssist (IBM/AAAI 2025) — trustworthiness mechanisms: positional bias checking, certainty estimation, token-probability judgment
- Arize Phoenix / MLflow / LangSmith comparison (2025) — dataset management, hallucination scorers, golden dataset versioning patterns
