# STACK: SQL Agent Değerlendirme Platformu

## Recommended Stack
- **Backend:** Python (FastAPI)
- **Database:** PostgreSQL (SQLAlchemy / psycopg)
- **Validation:** Pydantic
- **Eval Orchestration:** LangSmith (for trace/experiment) or a Custom Python Runner script initially.

## Rationale
Python is the standard for data/LLM pipelines. LangSmith provides native experimental / trace tracking out of the box. PostgreSQL provides solid storage for historical runs and golden metrics JSONs.

## Do Not Use
- Excel/CSV based storage for production eval, because it lacks traceability and relation schema constraints.
