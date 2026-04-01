# Architecture Patterns: SQL Evaluation & LLM-as-a-Judge Platform

**Domain:** SQL evaluation + LLM judge pipeline  
**Researched:** 2026-04-01  
**Confidence:** HIGH (based on project context analysis + verified patterns from pydantic-evals, instructor, DuckDB docs)

---

## Recommended Architecture

### Overall Pattern: Sequential Pipeline with Shared Context

The canonical pattern for this domain is a **Sequential Pipeline with Shared Evaluation Context** — not pure event-driven, not pure chain-of-responsibility. Each layer receives an immutable input context and enriches a mutable result context. Layers are ordered by computational cost and short-circuit on critical failures.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EvaluationPipeline                          │
│                                                                     │
│  EvalCase ──► Ingestion ──► Normalizer ──► EvalContext (mutable)    │
│                                                │                    │
│                          ┌─────────────────────▼────────────────┐  │
│                          │         Pipeline Stages               │  │
│                          │  1. RuleEngineEvaluator               │  │
│                          │  2. ExecutionValidator                │  │
│                          │  3. ASTSemanticValidator              │  │
│                          │  4. ResultSetComparator               │  │
│                          │  5. LLMJudge                          │  │
│                          │  6. JudgeQualityScorer                │  │
│                          └──────────────────────────────────────┘  │
│                                                │                    │
│                          ScoreAggregator ◄─────┘                    │
│                                │                                    │
│                          EvalResult (immutable final)               │
└─────────────────────────────────────────────────────────────────────┘
```

**Why pipeline over chain-of-responsibility:**
- Chain-of-responsibility implies each handler decides whether to pass forward — this is correct in principle but leads to scattered early-exit logic. In this pipeline, early-exit is explicit via a `critical_failure` flag on `EvalContext`, checked by the pipeline orchestrator at each stage boundary.
- Pipeline pattern keeps all routing logic in one place (the runner), all evaluation logic in the evaluators.

---

## Component Boundaries

| Component | Responsibility | Does NOT Do | Communicates With |
|-----------|---------------|-------------|-------------------|
| `Ingestion` | Load golden cases + agent outputs; validate schema | Score, judge, transform | `CaseRepository`, `AgentOutputLoader` |
| `Normalizer` | Normalize SQL whitespace, case, formatting | Execute SQL, evaluate semantics | `EvalContext` (write) |
| `RuleEngineEvaluator` | Check structural rules, risk patterns, business rules | Execute SQL, call LLM | `EvalContext` (read/write), `RuleRegistry` |
| `ExecutionValidator` | Execute candidate + gold SQL; measure execution accuracy | Parse AST, call LLM | `SQLExecutor` (protocol), `EvalContext` |
| `ASTSemanticValidator` | Parse SQL to AST; compare normalized AST trees | Execute SQL, call LLM | `ASTParser`, `EvalContext` |
| `ResultSetComparator` | Compare row/column/value outputs | Execute SQL, parse AST | `EvalContext` (reads execution results) |
| `LLMJudge` | Intent fidelity, business correctness, answer grounding, holistic decision | Score itself, persist | `LLMClient` (protocol), `PromptRegistry`, `EvalContext` |
| `JudgeQualityScorer` | Score judge output quality (specificity, completeness, rationale, actionability) | Call LLM directly | `EvalContext` (reads LLM judge output) |
| `ScoreAggregator` | Apply weights, check override rules, produce final decision | Store results, report | `EvalContext` → `EvalResult` |
| `ResultStore` | Persist `EvalResult` objects | Evaluate, transform, report | Storage backend (CSV/JSONL or PostgreSQL) |
| `ReportGenerator` | Compute batch metrics, detect regressions, render CLI output | Evaluate, store | `ResultStore`, `RegressionTracker` |
| `BatchRunner` | Orchestrate pipeline across all cases; manage concurrency | Evaluate directly | `EvaluationPipeline`, `ResultStore`, `ReportGenerator` |

---

## Data Flow Direction

```
INGESTION PHASE
  golden_eval_dataset.csv  ──────────────────────────────────────┐
  agent_output.json/csv    ──────────────────────────────────────┤
                                                                  ▼
                                                           EvalCase (dataclass)
                                                                  │
PIPELINE PHASE (per case)                                         ▼
  ┌─────────────────────────────────────────────────────── EvalContext ◄────────────┐
  │                                                               │                  │
  │   RuleEngineEvaluator ─────────────────────────────────────► │ rule_results      │
  │         │ (critical failure? → set flag, pipeline stops)      │                  │
  │         ▼                                                     │                  │
  │   ExecutionValidator ──────────────────────────────────────► │ execution_results │
  │         │ (uses SQLExecutor protocol → CSV or PG backend)     │                  │
  │         ▼                                                     │                  │
  │   ASTSemanticValidator ────────────────────────────────────► │ ast_results       │
  │         ▼                                                     │                  │
  │   ResultSetComparator ─────────────────────────────────────► │ result_results    │
  │         │ (reads execution_results, no new execution)         │                  │
  │         ▼                                                     │                  │
  │   LLMJudge ─────────────────────────────────────────────────► │ judge_output     │
  │         │ (structured JSON via Pydantic model)                │                  │
  │         ▼                                                     │                  │
  │   JudgeQualityScorer ──────────────────────────────────────► │ quality_scores    │
  │                                                               │                  │
  └─────────────────────────────────────────────────────────────►│ critical_failure  │
                                                                  │                  │
AGGREGATION PHASE                                                 ▼
  ScoreAggregator ◄───────────────────────────────────────── EvalContext (final)
       │ apply weights + override rules
       ▼
  EvalResult (immutable: decision, overall_score, all dimension scores, issues, observability)
       │
STORAGE PHASE                                                     
  ResultStore.save(EvalResult) ──► JSONL files (v1) / PostgreSQL (v2)
       │
REPORTING PHASE
  ReportGenerator.render(run_results) ──► terminal CLI output + regression list
```

**Critical design rule:** Data flows FORWARD only. No layer calls a previous layer. No layer reaches into storage directly (only `ResultStore` does). The `EvalContext` is the single mutable accumulator passed through the pipeline.

---

## Interface Contracts Between Layers

### 1. BaseEvaluator Protocol (all pipeline stages implement this)

```python
from typing import Protocol
from dataclasses import dataclass

class BaseEvaluator(Protocol):
    name: str                     # e.g. "rule_engine", "execution_validator"
    
    def evaluate(self, context: EvalContext) -> None:
        """
        Mutates context in-place by adding results to context.results[self.name].
        Sets context.critical_failure = True if a fatal condition is detected.
        MUST be idempotent — safe to call multiple times.
        MUST NOT call other evaluators or storage.
        """
        ...
    
    @property
    def can_short_circuit(self) -> bool:
        """If True, pipeline stops here on critical_failure from this layer."""
        ...
```

### 2. EvalContext (shared mutable accumulator)

```python
@dataclass
class EvalContext:
    # Inputs (immutable after creation)
    case: EvalCase
    candidate_sql: str
    candidate_answer: str
    candidate_result: list[dict]
    
    # Observability (written by ExecutionValidator and LLMJudge)
    observability: ObservabilityMetadata
    
    # Accumulated results (each evaluator adds its section)
    rule_results: RuleResults | None = None
    execution_results: ExecutionResults | None = None
    ast_results: ASTResults | None = None
    result_comparison: ResultComparisonOutput | None = None
    judge_output: JudgeOutput | None = None
    judge_quality: JudgeQualityOutput | None = None
    
    # Pipeline control
    critical_failure: bool = False
    critical_failure_reason: str | None = None
```

### 3. SQLExecutor Protocol (execution abstraction layer)

```python
class SQLExecutor(Protocol):
    def execute(self, sql: str) -> ExecutionResult:
        """
        Returns: ExecutionResult(
            status: "success" | "empty" | "error",
            rows: list[dict],
            row_count: int,
            column_names: list[str],
            execution_time_ms: float,
            error_message: str | None
        )
        """
        ...

# Concrete implementations:
# CSVSandboxExecutor  — uses DuckDB in-memory; reads CSV as virtual tables
# PostgreSQLExecutor  — connects to real PG; used in production phase
```

**DuckDB as CSV sandbox** (HIGH confidence — verified):
```python
import duckdb

class CSVSandboxExecutor:
    def __init__(self, csv_paths: dict[str, str]):
        # csv_paths: {"table_name": "/path/to/file.csv"}
        self.conn = duckdb.connect(":memory:")
        for table_name, path in csv_paths.items():
            self.conn.execute(
                f"CREATE VIEW {table_name} AS SELECT * FROM read_csv_auto('{path}')"
            )
    
    def execute(self, sql: str) -> ExecutionResult:
        ...
```

This approach means the same SQL is executable against CSV in dev and PostgreSQL in production — the abstraction layer is the only change.

### 4. LLMJudge Output Contract (structured JSON via Pydantic)

```python
from pydantic import BaseModel

class JudgeIssue(BaseModel):
    type: str           # e.g. "missing_filter", "wrong_grain"
    severity: Literal["critical", "high", "medium", "low"]
    evidence: str       # specific quote or reference, not vague
    dimension: str      # which evaluation dimension

class JudgeOutput(BaseModel):
    intent_fidelity_score: float        # 0.0–1.0
    business_correctness_score: float   # 0.0–1.0
    answer_grounding_score: float       # 0.0–1.0
    holistic_decision: Literal["approve", "rewrite", "human_review", "reject"]
    confidence: float                   # 0.0–1.0
    rewrite_guidance: str | None        # populated when decision != approve
    issues: list[JudgeIssue]
    reasoning: str                      # full chain-of-thought
    hallucination_flags: list[str]      # answer hallucination signals
    prompt_version: str                 # tracked for reproducibility
    model_name: str
```

**Implementation approach** (MEDIUM confidence — instructor library patterns):
- Use `instructor` library with any LLM provider for guaranteed Pydantic-validated output
- Fallback: `response_format={"type": "json_object"}` + manual Pydantic parse with retry
- System prompt includes schema as JSON Schema, plus few-shot examples per dimension

### 5. CaseRepository Interface

```python
class CaseRepository(Protocol):
    def load_all(self) -> list[EvalCase]: ...
    def load_by_bucket(self, bucket: str) -> list[EvalCase]: ...
    def load_by_ids(self, ids: list[str]) -> list[EvalCase]: ...
    def get_case(self, case_id: str) -> EvalCase: ...

# Concrete implementations:
# CSVCaseRepository — reads golden_eval_dataset.csv + enrichment JSONL
# PostgreSQLCaseRepository — reads golden_cases table (Phase 5+)
```

---

## Build Order (Phase Dependencies)

The build order is **strictly determined by interface dependencies**. Later layers can only be built after their upstream interfaces are stable.

```
WAVE 0 — Data Models & Core Contracts (no dependencies)
  ├── EvalCase dataclass + golden dataset schema
  ├── EvalContext dataclass
  ├── EvalResult dataclass  
  ├── BaseEvaluator protocol
  ├── SQLExecutor protocol
  └── CaseRepository protocol

WAVE 1 — Storage Backends & Dataset Loading (depend on Wave 0 models)
  ├── CSVCaseRepository (reads golden CSV → EvalCase objects)
  ├── AgentOutputLoader (reads agent JSON/CSV output)
  ├── CSVSandboxExecutor (DuckDB; depends on SQLExecutor protocol)
  └── ResultStore / JSONL writer

WAVE 2 — Deterministic Evaluators (depend on Wave 0+1)
  ├── Normalizer (stateless text transform)
  ├── RuleEngineEvaluator (pure logic, no I/O)
  ├── ASTSemanticValidator (uses sqlglot/sqlparse; no execution)
  └── ResultSetComparator (pure comparison logic; reads ExecutionResults)

WAVE 3 — Execution Validator (depends on Wave 1+2)
  └── ExecutionValidator (executes SQL via SQLExecutor; produces ExecutionResults)

    NOTE: ResultSetComparator is logically after ExecutionValidator in the pipeline
    but can be built in parallel since it only depends on the ExecutionResults TYPE
    from Wave 0, not the actual ExecutionValidator implementation.

WAVE 4 — LLM Layer (depends on Wave 0+2+3 outputs)
  ├── LLMClient abstraction (protocol + OpenAI/Anthropic implementations)
  ├── PromptRegistry (versioned prompt templates)
  ├── LLMJudge (uses LLMClient + PromptRegistry + EvalContext)
  └── JudgeQualityScorer (pure scoring against JudgeOutput schema)

WAVE 5 — Aggregation & Pipeline Orchestration (depends on Wave 2+3+4)
  ├── ScoreAggregator (weighted scoring + override rules)
  └── EvaluationPipeline (orchestrator; wires all evaluators in order)

WAVE 6 — Batch Running & Reporting (depends on Wave 1+5)
  ├── BatchRunner (calls pipeline per case; parallelism optional)
  ├── RegressionTracker (compares run results to baseline)
  └── ReportGenerator (CLI terminal output; reads ResultStore)

WAVE 7 — PostgreSQL Migration (depends on all prior)
  ├── PostgreSQLCaseRepository
  ├── PostgreSQLExecutor
  └── PostgreSQLResultStore
```

**Critical path:** Wave 0 → Wave 1 (CSVCaseRepository) → Wave 3 (ExecutionValidator) → Wave 4 (LLMJudge) → Wave 5 (Pipeline) → Wave 6 (BatchRunner). Everything else can be built in parallel within waves.

---

## Directory Structure

```
reviewer/
├── models/                      # WAVE 0 — All dataclasses and protocols
│   ├── eval_case.py             # EvalCase, GoldenCase, AgentOutput
│   ├── eval_context.py          # EvalContext (mutable pipeline state)
│   ├── eval_result.py           # EvalResult (immutable final output)
│   ├── judge_output.py          # JudgeOutput, JudgeIssue (Pydantic)
│   └── protocols.py             # BaseEvaluator, SQLExecutor, CaseRepository
│
├── storage/                     # WAVE 1+7 — Storage backends
│   ├── case_repository/
│   │   ├── csv_repository.py    # reads golden CSV
│   │   └── postgres_repository.py  # Phase 5+
│   ├── result_store/
│   │   ├── jsonl_store.py       # v1 storage: one JSONL file per run
│   │   └── postgres_store.py   # Phase 5+
│   └── agent_output_loader.py
│
├── executors/                   # WAVE 1 — SQL execution backends
│   ├── base.py                  # SQLExecutor protocol
│   ├── csv_sandbox.py           # DuckDB-based CSV execution
│   └── postgresql.py            # Production PG executor
│
├── evaluators/                  # WAVE 2+3+4 — Pipeline stages
│   ├── normalizer.py
│   ├── rule_engine/
│   │   ├── evaluator.py         # RuleEngineEvaluator
│   │   └── rules/               # Individual rule modules
│   │       ├── structural.py    # syntax, parse checks
│   │       ├── business.py      # domain-specific rules
│   │       └── risk.py          # LIMIT, CURRENT_DATE, NULL patterns
│   ├── execution_validator.py
│   ├── ast_semantic_validator.py
│   ├── result_set_comparator.py
│   ├── llm_judge/
│   │   ├── judge.py             # LLMJudge evaluator
│   │   ├── prompts/             # Versioned prompt templates
│   │   │   ├── v1_system.txt
│   │   │   └── v1_user.txt
│   │   └── llm_client.py        # LLMClient protocol + implementations
│   └── judge_quality_scorer.py
│
├── aggregator.py                # WAVE 5 — ScoreAggregator
├── pipeline.py                  # WAVE 5 — EvaluationPipeline orchestrator
│
├── runners/                     # WAVE 6
│   ├── batch_runner.py
│   └── regression_tracker.py
│
├── reporting/                   # WAVE 6
│   └── report_generator.py
│
└── cli.py                       # Entry point
```

---

## Key Architectural Decisions

### Decision 1: Pipeline Orchestrator Owns Early-Exit Logic

The `EvaluationPipeline` checks `context.critical_failure` after each stage. Individual evaluators set the flag but never decide whether to stop. This keeps routing in one place.

```python
class EvaluationPipeline:
    def __init__(self, evaluators: list[BaseEvaluator]):
        self.evaluators = evaluators   # ordered list; config-driven
    
    def run(self, case: EvalCase, agent_output: AgentOutput) -> EvalResult:
        context = EvalContext.from_inputs(case, agent_output)
        for evaluator in self.evaluators:
            evaluator.evaluate(context)
            if context.critical_failure and evaluator.can_short_circuit:
                break   # skip LLM judge if SQL is fundamentally broken
        return ScoreAggregator().aggregate(context)
```

Extending the pipeline = add a new `BaseEvaluator` implementation and insert it in the list. No existing code changes.

### Decision 2: DuckDB as the CSV Sandbox Executor

DuckDB runs fully in-process with no server, supports the same SQL syntax as PostgreSQL (with minor dialect differences), and directly queries CSV files as virtual tables. This means:
- No separate database setup for testing
- Same `SQLExecutor` protocol as production PostgreSQL
- Migration to PostgreSQL = swap one class, zero pipeline changes

**Caveat:** DuckDB has minor SQL dialect differences from PostgreSQL (e.g., `ILIKE` behavior, some window function edge cases). The `ASTSemanticValidator` should normalize dialect-specific syntax before comparing.

### Decision 3: Versioned Prompt Registry for Regression Attribution

Every LLM judge run must log `prompt_version`. The `PromptRegistry` loads prompts by version string:

```python
class PromptRegistry:
    def get(self, version: str) -> tuple[str, str]:  # (system, user template)
        ...
```

Regression detection compares `(model_name, prompt_version)` pairs across runs to attribute score changes to model updates vs. prompt changes.

### Decision 4: JudgeQualityScorer is Deterministic, Not LLM-Based

The JudgeQualityScorer scores the LLM judge's output quality using **heuristic rules**, not a second LLM call. This avoids infinite regression (judge-of-judge hallucination) and keeps cost predictable:

- **Specificity**: Does `evidence` field contain SQL snippet or column name? (regex/length check)
- **Completeness**: Number of issues flagged vs. expected for the decision type
- **Rationale density**: `reasoning` field word count vs. minimum threshold per decision
- **Actionability**: Does `rewrite_guidance` contain specific SQL clause references?

This is a deliberate architectural choice: the quality scorer is a structured linter of judge outputs, not another judge.

### Decision 5: Hallucination Detection as Cross-Cutting Signal

Hallucination is detected in two places but stored centrally:

- **Answer hallucination**: Detected by `LLMJudge` (semantic check: answer vs. result set scope). Judge populates `judge_output.hallucination_flags`.
- **Judge hallucination**: Detected by `JudgeQualityScorer` (checks if flagged issues reference columns/tables that don't exist in schema).
- Both write to `EvalContext.hallucination_events`, which `ScoreAggregator` reads to apply the critical override rule.

### Decision 6: ResultStore Abstraction Enables CSV→PostgreSQL Migration

```python
class ResultStore(Protocol):
    def save_result(self, result: EvalResult, run_id: str) -> None: ...
    def load_run(self, run_id: str) -> list[EvalResult]: ...
    def load_baseline(self) -> list[EvalResult]: ...

# v1: JSONLResultStore — one .jsonl file per batch run
# v2: PostgreSQLResultStore — reads/writes judge_scores, judge_issues, hallucination_events tables
```

`ReportGenerator` and `RegressionTracker` only know the `ResultStore` protocol; migration requires no changes to these components.

---

## Scalability Considerations

| Concern | At 968 cases (v1) | At 10K cases | At 100K cases |
|---------|-------------------|--------------|---------------|
| Batch execution | Sequential, ~30min with LLM | asyncio + semaphore | Worker pool + queue |
| LLM judge cost | ~$5-15/run (GPT-4o) | Cache identical prompts | Prompt-level caching + cheaper judge for easy cases |
| Storage | JSONL files | JSONL → PostgreSQL | PostgreSQL with partitioning |
| Regression detection | In-memory diff | Index on (case_id, run_date) | Time-series table with materialized views |
| CSV execution (DuckDB) | In-process per case | Shared DuckDB connection pool | DuckDB persistent file + parallel connections |

For 968 cases, asyncio concurrency with a semaphore (limit 10–20 concurrent LLM calls) brings total LLM judge time from ~30min sequential to ~3min. The deterministic layers (rule engine, execution, AST) run in milliseconds each and don't need concurrency optimization at this scale.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Evaluator Cross-Calls
**What:** `LLMJudge` calls `RuleEngineEvaluator` internally to re-check rules.
**Why bad:** Breaks single-direction data flow; makes testing harder; creates hidden coupling.
**Instead:** Pass accumulated `EvalContext` — judge reads `context.rule_results` set by the rule engine upstream.

### Anti-Pattern 2: Storage in Evaluators
**What:** `ExecutionValidator` writes execution results directly to CSV/DB inside `evaluate()`.
**Why bad:** Evaluators become stateful with I/O side effects; impossible to unit test without DB.
**Instead:** Evaluators only mutate `EvalContext`; `ResultStore` handles all persistence after pipeline completes.

### Anti-Pattern 3: Monolithic LLM Prompt
**What:** Single prompt asks LLM to evaluate all 8 dimensions at once.
**Why bad:** LLMs have limited attention for multi-objective tasks; harder to improve one dimension without affecting others; impossible to track which dimension caused a regression.
**Instead:** Prompt focuses on the 3 dimensions only LLM can evaluate (intent fidelity, business correctness, answer grounding). Structural/execution/result dimensions are handled by deterministic layers — their scores are passed to the LLM as context, not re-evaluated.

### Anti-Pattern 4: No Prompt Versioning
**What:** Prompt stored as a f-string in the judge class body.
**Why bad:** Changing the prompt invalidates historical regression comparisons; impossible to attribute score drift to model vs. prompt vs. data.
**Instead:** Every prompt is a versioned template file (`v1_system.txt`, `v2_system.txt`). `prompt_version` is always logged in `EvalResult`.

### Anti-Pattern 5: Second LLM for Judge Quality
**What:** Call a second LLM to evaluate if the judge output is good.
**Why bad:** Unbounded cost, introduces judge-of-judge hallucination, circular dependency.
**Instead:** `JudgeQualityScorer` uses deterministic heuristics (evidence specificity, reasoning length, rewrite guidance structure).

---

## Sources

- [Build a Production-Ready SQL Evaluation Engine for LLMs](https://dev.to/kasi_viswanath/build-a-production-ready-sql-evaluation-engine-for-llms-3e9k) — MEDIUM confidence (WebSearch)
- [LLM-as-a-Judge: A Practical Guide with Pydantic Evals](https://pydantic.dev/articles/llm-as-a-judge) — HIGH confidence (official)
- [Instructor — Structured LLM Outputs](https://python.useinstructor.com/) — HIGH confidence (official)
- [DuckDB Python API — In-Memory + CSV](https://duckdb.org/docs/stable/clients/python/dbapi) — HIGH confidence (official)
- [MLflow End-to-End Judge Workflow](https://mlflow.org/docs/latest/genai/eval-monitor/scorers/llm-judge/workflow/) — MEDIUM confidence (official)
- Project context: `plan_before.md` + `PROJECT.md` — HIGH confidence (primary source)
