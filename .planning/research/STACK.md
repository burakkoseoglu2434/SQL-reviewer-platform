# Technology Stack — SQL Evaluation & LLM-as-a-Judge Platform

**Project:** SQL Evaluation Pipeline + LLM Judge
**Researched:** 2026-04-01
**Overall confidence:** HIGH (all primary choices verified against PyPI, official docs, and official release notes)

---

## 1. SQL Parsing / AST Normalization

### Recommended: `sqlglot` v30.1.0

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| sqlglot | `>=25.0,<31` (pin to v30.x) | SQL parsing, AST traversal, dialect normalization | True parser (not tokenizer). Supports 31 SQL dialects, tree traversal, query fingerprinting, transpilation, and semantic equivalence checks. Pure Python; optional C extension (`sqlglot[c]`) for perf. |

**Confidence: HIGH** — 45.8M monthly PyPI downloads, 9,077 GitHub stars, last updated March 2026. Official docs at sqlglot.com.

**Why NOT sqlparse:**
sqlparse is a tokenizer, not a parser — it produces a flat token stream, not an AST. It cannot normalize column order, detect semantic equivalence, or handle dialect differences. The sqlglot authors explicitly built sqlglot to replace it.

**Key sqlglot features for this project:**
- `sqlglot.parse_one(sql)` → AST node tree
- `expr.sql(dialect="duckdb")` → normalized SQL string
- `sqlglot.diff(ast1, ast2)` → structural diff for semantic comparison
- `sqlglot.optimizer.normalize` → canonical form for equivalence checks

```bash
pip install "sqlglot[c]>=25.0"
```

---

## 2. Query Execution Sandbox (CSV/File-Based)

### Recommended: `duckdb` v1.5.1

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| duckdb | `>=1.5.0` | In-process SQL execution on CSV/DataFrame | Zero-config, in-memory or file-backed. Reads CSV/Parquet/JSON directly with `read_csv()`. Full SQL dialect. |

**Confidence: HIGH** — DuckDB 1.5.1 released March 23, 2026 (official release blog). LTS track (1.4.4) also available.

**Why DuckDB over alternatives:**
- **vs. SQLite**: DuckDB is OLAP-optimized (vectorized execution), handles analytical queries (aggregations, window functions) far faster. SQLite is OLTP row-by-row.
- **vs. pandas eval**: DuckDB runs standard SQL directly on DataFrames/CSVs without converting to pandas API. No impedance mismatch.
- **vs. spinning up PostgreSQL**: Zero server setup. Runs in-process. Identical SQL semantics as PostgreSQL for most queries.

**Key patterns for the evaluation sandbox:**
```python
import duckdb

# Execute candidate SQL against CSV data
conn = duckdb.connect()
conn.execute("CREATE TABLE orders AS SELECT * FROM read_csv('data.csv')")
result = conn.execute(candidate_sql).fetchdf()  # returns pandas DataFrame
```

**Migration note:** DuckDB's SQL dialect is largely PostgreSQL-compatible, so query execution tests will transfer cleanly when migrating to live PostgreSQL.

```bash
pip install "duckdb>=1.5.0"
```

---

## 3. LLM Integration + Structured Output

### Primary: Native OpenAI / Anthropic SDKs with Pydantic v2

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| openai | `>=1.60.0` | OpenAI API client | `client.beta.chat.completions.parse()` supports native Pydantic model output. gpt-4o and later guarantee schema adherence. |
| anthropic | `>=0.50.0` | Anthropic API client | `client.messages.parse()` with `output_format=PydanticModel`. Available on Claude Opus/Sonnet/Haiku 4.x series. |
| pydantic | `>=2.8` | Structured output models, validation | v2 is required for native SDK structured output. `BaseModel` fed directly to parse calls — SDK handles JSON schema generation + deserialization. |

**Confidence: HIGH** — Verified against official Anthropic docs (docs.anthropic.com/structured-outputs, Feb 2026) and OpenAI docs (developers.openai.com, Aug 2024+).

### Optional: `instructor` library for unified multi-provider interface

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| instructor | `>=1.4.0` | Unified structured output wrapper | Single API across OpenAI, Anthropic, Gemini, Mistral. Automatic retry on schema failures. Useful if the judge needs to be provider-agnostic. |

**Confidence: MEDIUM** — instructor is popular (production-grade) but adds a dependency layer. Use if you want to swap providers without code changes.

**Why NOT raw JSON mode / function calling:**
JSON mode only guarantees valid JSON, not schema compliance. Native structured output (Pydantic parse) guarantees type-correct fields. Use native unless you need a provider not yet supporting it.

**Judge output model example:**
```python
from pydantic import BaseModel, Field
from typing import Literal

class JudgeVerdict(BaseModel):
    verdict: Literal["pass", "fail", "partial"]
    score: float = Field(ge=0.0, le=1.0)
    reasoning: str
    hallucination_detected: bool
    confidence: Literal["high", "medium", "low"]
```

```bash
pip install "openai>=1.60.0" "anthropic>=0.50.0" "pydantic>=2.8"
# Optional:
pip install "instructor>=1.4.0"
```

---

## 4. Data Validation & Result Set Comparison

### Recommended: `pydantic` v2 + `pandas` + `numpy` (numeric tolerance)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pydantic | `>=2.8` | Input/output schema validation, judge models | Already required for LLM integration. Use `BaseModel` for all pipeline I/O contracts. |
| pandas | `>=2.2` | Result set loading, comparison, CSV I/O | Battle-tested for tabular data; integrates natively with DuckDB (`fetchdf()`). DataFrame-level comparison with `pd.testing.assert_frame_equal(check_exact=False, rtol=1e-3)`. |
| numpy | `>=1.26` | Numeric tolerance comparisons | `numpy.isclose(a, b, rtol=1e-3, atol=1e-5)` for element-wise float comparison. Already a pandas dependency. |

**Confidence: HIGH** — pandas 2.2 stable, numpy 1.26 stable. Both official PyPI.

**Why NOT polars for primary comparison:**
Polars has better numeric tolerance APIs (`assert_frame_equal` with rtol/atol), but DuckDB returns pandas DataFrames via `fetchdf()` natively. Adding polars as a second DataFrame library adds conversion overhead. Pandas is sufficient for a 968-row golden dataset.

**Optional:** Add polars later if dataset scales to millions of rows — polars is 5-10x faster on large aggregations.

**Result comparison pattern:**
```python
import pandas as pd
import numpy as np

def compare_result_sets(expected: pd.DataFrame, actual: pd.DataFrame, rtol=1e-3) -> bool:
    try:
        pd.testing.assert_frame_equal(
            expected.sort_values(by=expected.columns.tolist()).reset_index(drop=True),
            actual.sort_values(by=actual.columns.tolist()).reset_index(drop=True),
            check_like=True,       # ignore column order
            check_exact=False,
            rtol=rtol,
        )
        return True
    except AssertionError:
        return False
```

```bash
pip install "pandas>=2.2" "numpy>=1.26"
```

---

## 5. CLI Runner / Batch Processor

### Recommended: `typer` + `rich`

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| typer | `>=0.12` | CLI commands, arguments, subcommands | Built on Click; uses Python type hints to eliminate boilerplate. Auto-generates `--help`, tab completion. Ideal for `evaluate`, `report`, `compare-runs` subcommands. |
| rich | `>=14.0` | Terminal tables, progress bars, colored output | Pass-rate summaries, regression diffs, hallucination counts all rendered as Rich tables. Progress bar for batch runs. No effort to look professional. |

**Confidence: HIGH** — rich 14.3.3 on PyPI (March 2026), typer 0.12.x stable. Both widely adopted in CLI tooling.

**Why NOT argparse:** No auto-help generation, no type coercion, significant boilerplate. Fine for scripts; not for a multi-command eval tool.

**Why NOT Click directly:** Typer wraps Click. Use Typer for new builds (less code), drop to raw Click only if you hit an edge case Typer can't handle.

**CLI structure:**
```
evaluate run --input results.csv --golden golden.csv --output report.json
evaluate report --input report.json --format table
evaluate compare --run-a report_v1.json --run-b report_v2.json
```

```bash
pip install "typer>=0.12" "rich>=14.0"
```

---

## 6. Storage Layer

### Phase 1 (current): CSV/JSON files with `pydantic` serialization

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pydantic | `>=2.8` | Serialize/deserialize all pipeline state | `model.model_dump_json()` / `Model.model_validate_json()`. Zero extra dependencies. |
| python-dotenv | `>=1.0` | Load `.env` for API keys in dev | Lightweight. pydantic-settings reads `.env` automatically. |
| pydantic-settings | `>=2.3` | Typed config from env vars / `.env` | `BaseSettings` with `env_file=".env"`, type-safe. Replaces manual `os.getenv()` calls. Manages `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `DATABASE_URL`, etc. |

**Confidence: HIGH** — pydantic-settings 2.0+ documented at docs.pydantic.dev; pydantic-settings 2.3 on PyPI.

### Phase 2 (PostgreSQL migration): SQLAlchemy 2 + Alembic

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| sqlalchemy | `>=2.0` | ORM / Core for PostgreSQL | SQLAlchemy 2.0 is a major rewrite with async support. Use `DeclarativeBase` pattern. Most mature Python ORM. |
| alembic | `>=1.13` | Schema migrations | Standard companion to SQLAlchemy. `alembic revision --autogenerate` from models. |
| psycopg2-binary | `>=2.9` | PostgreSQL driver | Sync driver for SQLAlchemy. Switch to `asyncpg` when async needed. |

**Confidence: HIGH** — SQLAlchemy 2.0 official docs confirm Pydantic v2 integration via `from_attributes=True`. Alembic 1.13 stable.

**Why NOT SQLModel for the migration phase:**
SQLModel (Pydantic + SQLAlchemy in one model) has appeal but Pydantic v2 support only fully landed August 2025. For a phase-2 migration, keeping Pydantic domain models separate from SQLAlchemy persistence models is more maintainable and avoids SQLModel's historical lag in keeping up with both parent libraries.

**Why NOT raw psycopg2 without ORM:**
The evaluation pipeline will query results, aggregate scores, compare runs — these benefit from ORM-level abstractions. Raw SQL queries for everything creates maintenance burden.

```bash
# Phase 1
pip install "pydantic-settings>=2.3" "python-dotenv>=1.0"

# Phase 2 additions
pip install "sqlalchemy>=2.0" "alembic>=1.13" "psycopg2-binary>=2.9"
```

---

## 7. Testing Framework

### Recommended: `pytest` + `pytest-mock` + `responses` / `respx`

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pytest | `>=8.0` | Test runner, fixtures, parametrize | De facto standard. `@pytest.mark.parametrize` essential for golden dataset sampling. |
| pytest-mock | `>=3.14` | Mock LLM API calls | `mocker.patch()` for OpenAI/Anthropic clients. Prevents live API calls in CI. |
| responses | `>=0.25` | HTTP-level mock for OpenAI SDK | Intercepts `requests`-based HTTP calls. Use when mocking at SDK level isn't sufficient. |
| pytest-cov | `>=5.0` | Code coverage | `--cov=src --cov-report=html`. Keep evaluation logic (not LLM calls) at >80% coverage. |

**Confidence: HIGH** — pytest 8.x current stable, all libraries confirmed on PyPI.

**Testing strategy:**
- **Unit tests**: rule engine, AST normalizer, result set comparator — fully deterministic, no mocks needed
- **Integration tests**: DuckDB execution sandbox with real SQL fixtures
- **LLM judge tests**: mocked API responses with pre-recorded judge verdicts (golden fixtures)
- **Golden dataset regression**: parametrized test that runs the full pipeline on a 10% sample of the golden CSV and asserts pass-rate above threshold

```bash
pip install "pytest>=8.0" "pytest-mock>=3.14" "pytest-cov>=5.0" "responses>=0.25"
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| SQL Parsing | sqlglot | sqlparse | sqlparse is a tokenizer, not a true parser; no AST, no dialect support, no semantic comparison |
| SQL Parsing | sqlglot | mo_sql_parsing | Less active, fewer dialect targets, smaller community |
| Query Sandbox | duckdb | SQLite | SQLite is row-store OLTP; slower for analytical evaluation queries; less PostgreSQL-compatible |
| Query Sandbox | duckdb | Spinning up real PostgreSQL | Massive setup overhead for a dev/eval sandbox; DuckDB is 95% PostgreSQL-compatible |
| LLM Structured Output | Native SDK parse() | LangChain | LangChain adds 15+ transitive dependencies for a use case that native SDKs now handle natively |
| LLM Structured Output | Native SDK parse() | LlamaIndex | Same as LangChain — overkill for a judge integration |
| DataFrame | pandas | polars | polars is faster at scale but DuckDB returns pandas natively; polars adds conversion overhead for a 968-row dataset |
| CLI | typer | argparse | argparse requires manual help generation, no type coercion, excessive boilerplate |
| ORM | sqlalchemy 2 | tortoise-orm | tortoise is async-only; SQLAlchemy supports both sync and async, has larger ecosystem |
| ORM | sqlalchemy 2 | peewee | peewee has no SQLAlchemy 2.x-style DeclarativeBase; smaller ecosystem; less Alembic integration |

---

## Full Installation

```bash
# Core evaluation pipeline
pip install \
  "sqlglot[c]>=25.0" \
  "duckdb>=1.5.0" \
  "pandas>=2.2" \
  "numpy>=1.26" \
  "pydantic>=2.8" \
  "pydantic-settings>=2.3" \
  "openai>=1.60.0" \
  "anthropic>=0.50.0" \
  "typer>=0.12" \
  "rich>=14.0" \
  "python-dotenv>=1.0"

# Optional: unified LLM provider wrapper
pip install "instructor>=1.4.0"

# Phase 2: PostgreSQL storage
pip install "sqlalchemy>=2.0" "alembic>=1.13" "psycopg2-binary>=2.9"

# Dev/test
pip install "pytest>=8.0" "pytest-mock>=3.14" "pytest-cov>=5.0" "responses>=0.25"
```

---

## Confidence Summary

| Area | Level | Source |
|------|-------|--------|
| sqlglot (SQL parsing) | HIGH | PyPI v30.1.0 confirmed; GitHub active March 2026 |
| duckdb (execution sandbox) | HIGH | Official release blog v1.5.1, March 23 2026 |
| OpenAI structured output | HIGH | Official OpenAI API docs, gpt-4o-2024-08-06+ |
| Anthropic structured output | HIGH | Official Anthropic docs, claude-4.x models, Feb 2026 |
| pydantic v2 | HIGH | docs.pydantic.dev, v2.8 stable |
| pandas result comparison | HIGH | PyPI 2.2, pandas testing module docs |
| typer + rich | HIGH | PyPI confirmed; rich v14.3.3, typer 0.12.x |
| SQLAlchemy 2 + Alembic | HIGH | Official SQLAlchemy docs; Pydantic v2 integration confirmed |
| pydantic-settings | HIGH | docs.pydantic.dev, v2.3 stable |
| pytest stack | HIGH | PyPI confirmed, de facto standard |
| instructor | MEDIUM | Popular library; adds dependency layer; not strictly necessary if single-provider |

---

## Sources

- sqlglot: https://pypi.org/project/sqlglot/ (v30.1.0, March 2026)
- duckdb release: https://duckdb.org/2026/03/23/announcing-duckdb-151.html
- OpenAI structured outputs: https://developers.openai.com/docs/guides/structured-outputs
- Anthropic structured outputs: https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs
- pydantic settings: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- rich: https://pypi.org/project/rich/ (v14.3.3)
- SQLAlchemy + Pydantic v2: https://building.theatlantic.com/pydantic-v2-sqlalchemy-alembic-the-clean-future-proof-way-to-make-them-work-together-9325a2e43d4a
