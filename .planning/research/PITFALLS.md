# PITFALLS: SQL Agent Değerlendirme Platformu

## 1. Golden SQL Overfitting
- **Sign:** System starts rejecting fully working queries because they don't match AST of Gold SQL exactly.
- **Prevention:** Do not rely on strictly string matching. Rely on Result Equivalency (Sandbox Exec) + Semantic equivalence (LLM).

## 2. Unsafe Execution Pipeline
- **Sign:** Agent writes a destructive query that somehow passes the read-only or touches live data.
- **Prevention:** Use extreme isolation, strict read-only user, timeout threshold, row limitation boundaries.

## 3. Ambiguous Metric Business Logic
- **Sign:** Human reviewers disagree with LLM score.
- **Prevention:** Create a "Metric Dictionary". Supply exactly this dictionary to the LLM during evaluation prompt.
