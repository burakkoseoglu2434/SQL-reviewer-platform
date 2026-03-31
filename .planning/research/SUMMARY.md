# SUMMARY: SQL Agent Değerlendirme Platformu Research Synthesized

## Core Verdict
Building a multi-layered execution + LLM Evaluation platform using Python & PostgreSQL. Rule-based evaluation handles explicit deterministic bugs while LLM acts as the ultimate tie-breaker/semantic judge based on sandbox context.

## Stack
- Python / FastAPI
- PostgreSQL / SQLAlchemy
- LangSmith (Eval Orchestrator)

## Critical Path
1. **Infrastructure**: Sandbox + Offline DB for Golden Metrics.
2. **Parallel Validation**: Build Rule, Sandbox Exec, and Strict LLM evaluator pipelines that run asynchronously.
3. **Composite Integration**: Unifying three different evaluations into one JSON threshold decision (Approve/Reject).

## Top Risks to Mitigate
- Execution side defects (Sandbox DB dropping out or being unsafe).
- Relying exclusively on syntax over execution equivalency.
- Halüsinasyon in the LLM Judge step. Needs strong structured JSON enforcement.
