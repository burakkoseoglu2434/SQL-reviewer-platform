# ARCHITECTURE: SQL Agent Değerlendirme Platformu

## Components
1. **Ingestion Service**: Pulls Golden Dataset + Agent Log Traces.
2. **Normalization Service**: Cleans up SQL queries, standardizes whitespace/casing.
3. **Rule Engine**: Deterministic Python pipeline checking parsing, filters, columns.
4. **Execution Validator**: Runs candidate and gold SQL on a Sandbox Read-Only schema and returns diffs.
5. **LLM Judge Service**: Processes all generated context arrays, formats output as JSON using provided strict schema prompt.
6. **Eval Orchestrator**: Aggregates component scores, calculates Final/Composite Score.
7. **Reporting Dashboard / UI**: Visualizes the PostgreSQL rows to show top regressions, error taxonomy.

## Sequence Flow
Trace/Golden SQL -> Normalized -> (Parallel: Rule Validator, Execution Validator) -> Combined Context -> LLM Judge -> Evaluated JSON -> Final Decision Matrix -> Return Approve/Human Review/Reject.
