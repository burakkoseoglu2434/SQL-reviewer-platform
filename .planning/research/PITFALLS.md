# Domain Pitfalls: SQL Evaluation & LLM-as-a-Judge Platform

**Domain:** SQL evaluation + LLM-as-a-judge pipeline
**Researched:** 2026-04-01
**Overall confidence:** HIGH (cross-verified with academic sources, production post-mortems, and official docs)

---

## Already Mitigated (Acknowledged in PROJECT.md)

The following risks are already addressed by the current design — listed here only to prevent re-opening:

| Risk | Mitigation in Place |
|------|---------------------|
| Gold SQL is not the only correct answer | Result equivalence + AST normalization |
| Judge hallucination | Structured output + evidence requirement + judge_quality_score |
| Product drift | Monthly review + product_semantics_version |
| Overfitting to golden set | Adversarial dataset bucket |
| Empty result false positive | non_empty_execution_accuracy as primary metric |
| Numeric tolerance | atol=1e-5 |

The pitfalls below are **additional risks not yet addressed** by the current design.

---

## Critical Pitfalls

Mistakes that cause silent evaluation failure, corrupt metrics, or require pipeline rewrites.

---

### Pitfall C-1: LLM Judge Position Bias (Order Effect)

**What goes wrong:** When presenting candidate SQL and gold SQL side-by-side in the prompt, LLM judges systematically favor whichever appears first — regardless of actual quality. A 2025 systematic study across 15 LLM judges and 150,000 evaluation instances confirmed this is not random chance; it is structurally driven by autoregressive attention that gives earlier tokens higher salience.

**Why it happens:** Autoregressive language models process tokens sequentially. The first presented candidate receives disproportionate attention weight. The model then rationalizes its position-anchored preference in its explanation.

**Consequences:**
- Evaluation scores shift with prompt ordering
- A/B comparisons of model versions return inconsistent results
- Judge scores are non-reproducible across run orderings
- Regression detection breaks (same SQL scores differently month-to-month)

**Warning signs:**
- Judge scores change when you reverse the order of candidate vs. gold in the prompt
- High variance in judge scores for semantically identical inputs across runs
- Model A beats Model B when presented first, B beats A when second

**Prevention:**
1. Always run each case in **both orderings** (candidate-first and gold-first) and average scores
2. Alternatively, use a **single-item scoring** format where the judge scores the candidate independently without seeing gold SQL in the same prompt turn
3. Use structured rubrics with explicit criteria headers to anchor the judge to content, not position
4. Low temperature (≤ 0.3) reduces but does not eliminate this bias

**Phase to address:** Phase 1 (LLM Judge) — bake into prompt design before first run

---

### Pitfall C-2: LLM Judge Verbosity Bias (Length Heuristic)

**What goes wrong:** LLM judges assign higher scores to longer, more elaborate SQL or longer explanations, independent of correctness. A verbose-but-wrong candidate SQL can outscore a terse-but-correct one. In the context of judge output quality scoring, a judge that produces longer explanations scores higher on `specificity` and `rationale_density` even when the extra words are padding.

**Why it happens:** LLMs are trained on human preference data where humans often conflate length with effort and quality. This bias is baked into RLHF-trained models.

**Consequences:**
- Candidate SQL with verbose CTEs and comments scores artificially high
- Rewrite guidance that pads word count is rated more actionable than terse, precise guidance
- `judge_quality_score` inflated for verbose-but-vague explanations

**Warning signs:**
- Shorter gold SQL routinely scores lower than longer equivalent candidate SQL
- Judge explanations that say the same thing in many ways score as more "complete"
- Positive correlation between `output_tokens` and `judge_quality_score`

**Prevention:**
1. Include explicit anti-verbosity instruction in judge prompt: _"Longer is not better. Score only on correctness, not on length."_
2. Track `output_tokens` as a covariate when monitoring `judge_quality_score` drift
3. Include short-but-correct adversarial cases in the golden set that should score highly
4. Separate `specificity` into _precision_ (is the claim specific?) vs. _length_ (is it long?)

**Phase to address:** Phase 1 (LLM Judge) — prompt engineering; Phase 3 (Judge Quality Scorer) — covariate tracking

---

### Pitfall C-3: Judge Self-Enhancement / Family Bias

**What goes wrong:** When the LLM judge is the same model (or same model family) as the agent being evaluated, it assigns systematically higher scores to its own outputs. AWS research confirmed GPT-4o and Claude 3.5 Sonnet both exhibit self-bias. Family bias extends this: GPT-4o-judge inflates scores for GPT-4o-mini-generated SQL.

**Why it happens:** Models converge on similar stylistic preferences and implicit assumptions during training. A judge and an agent from the same family share failure modes — the judge cannot penalize what it cannot detect.

**Consequences:**
- Evaluation metrics are inflated when judge and agent share a model family
- True regression rates are underreported
- Switching model families reveals previously invisible failure modes

**Warning signs:**
- Judge approval rates drop when you switch to a different-family judge
- Adversarial cases designed for one model family are scored as "borderline" by a judge from the same family

**Prevention:**
1. Use a **different model family** for the LLM judge than for the SQL agent being evaluated
2. Run periodic **cross-judge calibration**: score the same batch with two different judge families; flag cases with ≥ 0.2 score divergence for human review
3. Document `judge_model_name` and `agent_model_name` in every run's observability metadata (already planned for `model_name` — extend to `judge_model_name`)

**Phase to address:** Phase 1 (LLM Judge) — model selection; Phase 3 (Observability) — cross-judge tracking

---

### Pitfall C-4: Composite Score Masking Critical Failures (Goodhart's Law)

**What goes wrong:** A SQL query that fails completely on one critical dimension can still produce a passing composite score (≥ 0.85) if it scores perfectly on all other dimensions. For example: a SQL that returns the right result set but for completely wrong business reasons (lucky coincidence) can achieve approve status. Or a SQL that hallucinates a metric definition but executes correctly scores high on execution and result_equivalence, masking the business_correctness failure.

**Why it happens:** Weighted average scoring smooths over sharp failures. With 7 dimensions, a 0.0 on one 15%-weighted dimension reduces composite by only 0.15 — not enough to cross a threshold.

**Consequences:**
- Catastrophic semantic errors pass the evaluation pipeline
- Business teams receive "approved" SQL that is actually wrong for their use case
- Over time, the model learns to game the dimensions with higher weights while neglecting lower-weight ones

**Warning signs:**
- High composite scores on cases that human reviewers flag as wrong
- `business_correctness` = 0.0 cases appearing in the approve bucket
- Flat improvement in composite score despite obvious qualitative regression in one dimension

**Prevention:**
1. **Dimension floor rules**: Any single critical dimension below a threshold should force `human_review` regardless of composite — not just the current execution-error override. Proposed thresholds: `intent_fidelity < 0.4` → human_review; `business_correctness < 0.3` → human_review
2. **Critical override extension**: The current override rules cover execution errors and parse failures. Extend to: `intent_fidelity ≤ 0.2` → force reject (not just execution_status = error)
3. **Dimension distribution report**: CLI report must show per-dimension score distributions, not just composite averages. A bimodal distribution (many 1.0s and 0.0s) reveals gaming
4. **Adversarial calibration cases**: Include golden cases specifically designed where a locally high score on one dimension should NOT compensate for failure on another

**Phase to address:** Phase 1 (Scoring model) — dimension floors; Phase 2 (Reporting) — dimension distribution output

---

### Pitfall C-5: AST Normalization False Equivalence (Dialect + Semantics Loss)

**What goes wrong:** AST normalization libraries (e.g., sqlglot) can both over-normalize (treating semantically different queries as equivalent) and under-normalize (treating syntactically different but equivalent queries as different). Specific known failure modes:
- `OUTER APPLY` (T-SQL) is misinterpreted as a missing JOIN, producing false positive "error" flags
- `normalize_joins` strips `LEFT ANTI JOIN` (Spark) to `LEFT JOIN`, losing semantic difference
- Adding table aliases or fully-qualifying column names is flagged as semantically different (false diff)
- Subquery vs. CTE rewrite of same logic is flagged as different

**Why it happens:** SQL has no formal canonical form across dialects. Normalization algorithms make local choices that are correct for one dialect and wrong for another.

**Consequences:**
- Correct SQL rejected as "semantically different" (false negative)
- Wrong SQL approved as "semantically equivalent" (false positive)
- Regression tests become noisy; developers lose trust in AST layer
- `semantic_score` dimension is unreliable → distorts composite

**Warning signs:**
- Human reviewers disagree with AST layer verdict on borderline cases
- Table alias changes (e.g., `a.col` vs `col`) cause semantic_score drops
- High false positive rate in the AST layer for CTE-vs-subquery rewrites

**Prevention:**
1. Define a **canonical SQL form** for your specific database dialect before implementing AST comparison — do not use sqlglot's default normalization settings
2. Use **result-set equivalence as the ground truth** for semantic equivalence; use AST comparison only as a fast pre-filter that can produce `needs_verification` but never final `reject`
3. Add dialect annotation to each golden case (`dialect: tsql | postgres | duckdb`) and configure the AST parser per case
4. Maintain a **normalization exception registry**: known patterns (alias normalization, CTE expansion) that should pass as equivalent
5. Test AST layer against adversarial SQL pairs before production: (a) same result, different syntax; (b) different result, similar syntax

**Phase to address:** Phase 1 (AST Validator) — normalization design; Phase 0 (Dataset) — add dialect annotation to golden cases

---

## Moderate Pitfalls

---

### Pitfall M-1: SQL Sandbox Side Effects and Injection

**What goes wrong:** When executing candidate SQL in a CSV/DuckDB sandbox, malicious or poorly formed queries can: (1) write files to disk via `COPY TO`, (2) execute shell commands via DuckDB's `read_csv_auto` with crafted paths, (3) exhaust memory on Cartesian joins, (4) use `SLEEP()`-equivalent constructs to cause timeouts that mask execution errors. A 2025 arxiv paper (ToxicSQL) demonstrated 79.41% attack success rates on LLM text-to-SQL systems via poisoned training data that generates executable-but-malicious SQL.

**Why it happens:** SQL sandboxing for evaluation is typically built for correctness testing, not security. The assumption that inputs are benign breaks under adversarial evaluation.

**Consequences:**
- `execution_status = success` for queries that have destructive side effects
- Timeout attacks make all queries appear as `execution_status = timeout`, degrading `execution_score` across the board
- Memory exhaustion from Cartesian products crashes the evaluation runner mid-batch

**Warning signs:**
- Candidate SQL contains `COPY`, `EXPORT`, `read_csv_auto`, or file path strings
- Evaluation runs that take significantly longer than expected on specific cases
- Memory usage spikes on specific golden cases

**Prevention:**
1. **Allowlist SQL statement types**: Only `SELECT` statements should be executable in evaluation. Parse the AST first; if root node is not `SELECT`, block execution and force `structural_error`
2. **Timeout enforcement**: Wrap every SQL execution in `signal.alarm()` (Unix) or `concurrent.futures.ThreadPoolExecutor` with `timeout` parameter; treat timeout as `execution_status = timeout` (not success)
3. **Memory cap**: For DuckDB, set `memory_limit` to a safe ceiling per query (e.g., 512MB); for pandas, check result row count before materializing
4. **Read-only connection**: Open the sandbox database in read-only mode; DuckDB supports `duckdb.connect(path, read_only=True)`
5. **Cardinality guard**: Before execution, check `EXPLAIN` output for estimated row count; abort queries with estimated output > 10M rows

**Phase to address:** Phase 1 (Execution Validator) — sandbox hardening

---

### Pitfall M-2: Golden Dataset Label Rot (Annotation Decay)

**What goes wrong:** Golden dataset labels become incorrect over time due to: (1) product/schema changes that make previously correct SQL wrong; (2) business rule changes that change what the "right" answer is; (3) gold SQL that was always slightly wrong but nobody caught it. The BIRD benchmark was found to have a 52.8% annotation error rate in a 2026 CIDR paper — re-evaluation of top models showed ranking shifts of up to 3 positions.

**Why it happens:** Golden datasets are created at a point in time. They are rarely re-validated systematically. Cases that were correct 6 months ago may have drifted silently.

**Consequences:**
- False negatives: correct candidate SQL is penalized because gold SQL is stale
- False positives: incorrect candidate SQL passes because gold SQL was already wrong
- Monthly reports show artificial metric drift that is actually dataset rot, not model regression
- Adversarial cases that were designed to catch specific failure modes now catch nothing because the failure mode was fixed but the gold label wasn't updated

**Warning signs:**
- Gold SQL execution errors during monthly replay (gold_execution_status = error)
- Cases where human reviewers consistently disagree with automated verdicts
- Gold row counts that no longer match expected ranges (e.g., gold returns 0 rows for a "positive" case)
- High rate of `borderline` cases that all cluster in the same domain or time period

**Prevention:**
1. **Re-execute gold SQL on every monthly run** and store `gold_execution_status` + `gold_row_count`. Flag any case where gold now returns empty or error as `needs_re-annotation`
2. **Label freshness tracking**: Add `last_validated_at` timestamp to each golden case (already in schema as `last_reviewed_at` — ensure it is actively updated, not just set at creation)
3. **Automatic drift alert**: If `gold_result_fingerprint` changes between monthly runs, treat case as `stale` and exclude from metrics until re-annotated
4. **Reviewer assignment**: Assign a human reviewer to all cases flagged as stale before the monthly report is published
5. **Confidence-weighted scoring**: Until a case is re-validated after a product change, downweight it in aggregate metrics

**Phase to address:** Phase 0 (Golden Dataset) — freshness schema; Phase 4 (Monthly Governance) — re-execution workflow

---

### Pitfall M-3: LLM Judge Anchoring Bias (Exemplar Contamination)

**What goes wrong:** If the judge prompt includes example cases (few-shot examples) or a reference score, the judge anchors its entire score distribution around those examples. A 2025 arxiv study confirmed anchoring effects shift entire output distributions and vary with model scale. In practice: if your few-shot examples all show `business_correctness = 0.8`, the judge clusters all outputs near 0.8 regardless of actual quality, compressing score variance below the threshold range needed for reliable approve/reject decisions.

**Why it happens:** LLM responses are conditioned on all prior context. Examples in the prompt function as implicit calibration anchors that the model does not consciously override.

**Consequences:**
- Score distribution collapses around anchor values → approve/reject thresholds lose discriminative power
- Low-quality SQL scores near 0.8 when anchor examples cluster near 0.8
- Month-to-month comparison is contaminated by anchor drift (different few-shot examples → different score scale)

**Warning signs:**
- Score distribution histogram shows artificial spike near specific values (e.g., 0.75, 0.80)
- Adding or changing few-shot examples shifts the entire score distribution
- Score variance drops significantly compared to early calibration runs

**Prevention:**
1. **Deliberately diverse anchors**: If using few-shot examples, include cases spanning the full range (0.1, 0.4, 0.7, 0.9, 1.0) rather than clustering in one tier
2. **Zero-shot preferred**: Consider using zero-shot prompting with a detailed rubric rather than few-shot examples; rubric quality is more controllable than example selection
3. **Calibration set**: Maintain a 20-case calibration set with known human-labeled scores; run it at the start of every monthly batch to detect judge score drift before publishing results
4. **Score distribution monitoring**: Track score distribution statistics (mean, std, skew, kurtosis) per dimension per run; alert if distribution shifts > 0.1 std from baseline

**Phase to address:** Phase 1 (LLM Judge) — prompt design; Phase 3 (Observability) — distribution monitoring

---

### Pitfall M-4: Prompt Version Drift Causing Silent Regression Attribution Errors

**What goes wrong:** When a monthly evaluation shows metric regression, the team investigates model changes. But if the judge prompt also changed between runs (even slightly — a rephrased sentence, added example), the regression may be entirely due to prompt change, not model change. Without strict prompt versioning, these are indistinguishable. Production LLM APIs also silently update models (GPT-4 in March 2023 behaved differently by June 2023 under the same version string).

**Why it happens:** Prompts are edited informally (copy in Notion, pasted into code). API model aliases like `gpt-4o` are updated silently by providers. There is no diff trail for prompt changes.

**Consequences:**
- Regressions attributed to model when they are prompt-caused (or vice versa)
- Months of evaluation history become incomparable
- A/B model comparison is corrupted if judge prompt changes mid-experiment

**Warning signs:**
- Score distribution shifts on cases that have not changed, coinciding with any prompt "cleanup"
- Regression list contains cases that were passing even before the alleged model change
- Evaluation scores on a pinned golden subset drift without any code change

**Prevention:**
1. **Hash the judge prompt**: Compute SHA-256 of the full rendered judge prompt template (not just the template file) and store it as `judge_prompt_hash` in each run's observability metadata. This is currently planned as `prompt_version` — extend to include a content hash, not just a version label
2. **Semantic versioning for prompts**: MAJOR change = changes decision logic or score weights; MINOR = adds examples or clarifies; PATCH = typo/grammar. Store in `CHANGELOG.md` in the prompt directory
3. **Pin API model snapshots**: Use dated model snapshots (`gpt-4o-2024-11-20`, `claude-3-5-sonnet-20241022`) instead of floating aliases. Document which snapshot was used for each monthly run
4. **Prompt change quarantine**: Any prompt change must be validated against the full golden set before being used for an official monthly run. The first run after a prompt change should be labeled `calibration_run = true` and excluded from trend comparisons

**Phase to address:** Phase 1 (LLM Judge) — prompt storage; Phase 3 (Observability) — hash tracking; Phase 4 (Governance) — change process

---

### Pitfall M-5: Execution Accuracy Inflation from Schema Mismatch Blindness

**What goes wrong:** In a CSV-based sandbox, the execution validator compares candidate SQL results against gold SQL results by running both against the same data snapshot. But if the candidate SQL references a different column or table that happens to exist in the sandbox (due to schema overlap or naming coincidence), it executes successfully and even returns rows — but against the wrong concept. The execution passes, `execution_score` is high, but `routing_accuracy = 0` and `business_correctness = 0`. The composite score is still misleadingly high.

**Why it happens:** CSV-based sandboxes contain all tables flat. A SQL that joins `products` instead of `product_hierarchy` may return overlapping data purely by coincidence. The execution validator cannot distinguish intentional schema usage from accidental schema overlap.

**Consequences:**
- `non_empty_execution_accuracy` inflated for semantically wrong SQL
- Routing errors (wrong table selection) are invisible to the execution layer
- Human reviewers catch errors that the evaluation pipeline approves

**Warning signs:**
- `routing_accuracy = 0` cases that still have `execution_score ≥ 0.7`
- Candidate SQL references tables not listed in the business_context expected schema
- Cases where result row counts match gold but column semantics are wrong

**Prevention:**
1. **Table reference validation**: Before execution, extract all table references from candidate SQL AST and verify each is in the expected schema set for that question (store `expected_tables` in golden case business_context)
2. **Routing accuracy as a pre-filter**: If `routing_accuracy = 0` (wrong table selected), cap `execution_score` to a maximum of 0.3 regardless of result match — don't allow result coincidence to compensate for schema error
3. **Column provenance tracking**: Extend result comparator to check that matched columns come from the same source table, not just share the same name

**Phase to address:** Phase 1 (Execution Validator + Rule Engine) — routing guard

---

## Minor Pitfalls

---

### Pitfall m-1: Score Calibration Ceiling Compression

**What goes wrong:** LLM judges using absolute scoring (0.0 to 1.0) tend to cluster scores in the 0.6–0.9 range, avoiding the extremes. A case that deserves a 0.2 is scored 0.5; a case that deserves 1.0 is scored 0.85. This compresses the discrimination range exactly where the approve/rewrite/human_review thresholds sit (0.65 and 0.85).

**Prevention:** Use calibration anchors at extreme ends (0.0 example: completely wrong SQL; 1.0 example: perfect SQL with no caveats) in the judge prompt. Track score distribution histogram; alert if < 5% of cases fall below 0.4 or above 0.9.

**Phase to address:** Phase 1 (LLM Judge) — calibration; Phase 3 (Observability) — distribution alert

---

### Pitfall m-2: NULL Handling Divergence Between Gold and Candidate

**What goes wrong:** SQL NULL semantics differ between flavors. A gold SQL using `COALESCE(col, 0)` and a candidate SQL using `col` (leaving NULLs) will produce different result sets that the result comparator will flag as non-equivalent — but from a business perspective, both may be acceptable depending on context. Conversely, a candidate that treats NULL as 0 when the business expects NULL to be excluded will silently pass numeric comparison within atol=1e-5.

**Prevention:** Add NULL handling as an explicit check in the result comparator: count NULL cells per column in both result sets and flag divergence > 5% as a warning issue (not an error override). Include NULL-heavy cases in adversarial bucket.

**Phase to address:** Phase 1 (Result Set Comparator)

---

### Pitfall m-3: Token Cost Explosion in LLM Judge at Scale

**What goes wrong:** With 968 golden cases, each LLM judge call costs ~500–1500 tokens in + 200–400 tokens out. At scale (monthly replay + model comparison runs), token costs accumulate rapidly. If the judge is called for ALL cases rather than only cases that passed deterministic layers, costs are 3–5x higher than necessary.

**Prevention:**
1. Use deterministic layers (rule engine + execution validator) as gatekeepers: only invoke the LLM judge when rule engine passes AND execution validates
2. Track `token_cost_per_run` in `monthly_reports` — alert if cost exceeds 2x the baseline from the first production run
3. Consider a smaller, cheaper model for first-pass LLM judgment with a larger model as second-pass for disputed cases

**Phase to address:** Phase 2 (Batch Runner) — judge routing logic; Phase 3 (Observability) — cost tracking

---

### Pitfall m-4: Adversarial Cases Becoming Stale (Red Team Decay)

**What goes wrong:** Adversarial cases are designed to catch specific failure modes of the current agent. As the agent improves, it learns to handle the original adversarial patterns. The adversarial bucket slowly migrates from "hard" to "trivial" — the overall pass rate rises not because of genuine improvement but because the agent was implicitly trained on the evaluation challenges.

**Prevention:**
1. Tag each adversarial case with `adversarial_target: [failure_mode_it_was_designed_to_catch]`
2. During monthly governance, if > 80% of adversarial cases in a given failure mode category now pass, flag the category for new adversarial case generation
3. Never use adversarial cases as training signal for the agent — evaluation contamination destroys the red team's value

**Phase to address:** Phase 4 (Monthly Governance) — adversarial refresh protocol

---

### Pitfall m-5: Report Metric Averaging Masking Bucket-Level Failures

**What goes wrong:** Global `pass_rate` and `non_empty_execution_accuracy` can look healthy while one specific bucket (e.g., `adversarial` or `borderline`) has catastrophically low performance. Monthly reports that aggregate across all cases hide bucket-level regressions.

**Prevention:** CLI report must always include per-bucket breakdown: `pass_rate_by_bucket`, `non_empty_execution_accuracy_by_bucket`. Any bucket where pass rate drops > 10 percentage points from the previous month triggers a mandatory regression investigation before the run is accepted.

**Phase to address:** Phase 2 (Reporting) — bucket-stratified metrics

---

## Phase-Specific Warning Map

| Phase | Phase Topic | Highest-Risk Pitfall | Mitigation Priority |
|-------|-------------|---------------------|---------------------|
| Phase 0 | Golden Dataset creation | M-2 (Label Rot) — labels wrong at creation | Add `last_validated_at`, re-execute gold SQL before publishing dataset |
| Phase 0 | Dataset schema | C-5 (AST dialect) — no dialect annotation | Add `dialect` field to each golden case now |
| Phase 1 | LLM Judge prompt design | C-1 (Position Bias) | Dual-ordering or single-item scoring from day one |
| Phase 1 | LLM Judge prompt design | C-2 (Verbosity Bias) | Explicit anti-verbosity instruction in prompt |
| Phase 1 | Judge model selection | C-3 (Self-Enhancement) | Select judge from different model family than agent |
| Phase 1 | Scoring model | C-4 (Composite masking) | Dimension floor rules before first scoring run |
| Phase 1 | Execution Validator | M-1 (Sandbox injection) | SELECT-only allowlist + timeout + read-only connection |
| Phase 1 | Execution Validator | M-5 (Schema mismatch) | Routing accuracy as execution cap |
| Phase 1 | AST Validator | C-5 (Normalization false equivalence) | Define canonical form per dialect; result-set as ground truth |
| Phase 2 | Reporting | m-5 (Metric averaging) | Per-bucket breakdown mandatory |
| Phase 2 | Batch Runner | m-3 (Token cost) | Deterministic gating before LLM judge |
| Phase 3 | Observability | M-4 (Prompt drift) | Store `judge_prompt_hash` in every run |
| Phase 3 | Observability | M-3 (Anchoring) | Score distribution monitoring alerts |
| Phase 4 | Monthly Governance | M-2 (Label Rot) | Re-execute gold SQL; flag stale fingerprints |
| Phase 4 | Monthly Governance | m-4 (Adversarial decay) | Tag adversarial targets; refresh trigger at 80% pass rate |

---

## Sources

- Zheng et al. (2025). "Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge." ACL Anthology IJCNLP 2025. https://aclanthology.org/2025.ijcnlp-long.18/
- AWS Research (2025). "Self-Bias and Family Bias in LLM-as-a-Judge." Statistical framework for self-enhancement evaluation.
- CalibraEval (2025). "Calibrating Prediction Distribution to Mitigate Selection Bias in LLMs-as-Judges." ACL 2025 Long. https://aclanthology.org/2025.acl-long.808/
- Anchoring Bias (2025). "Anchors in the Machine: Behavioral and Attributional Evidence of Anchoring Bias in LLMs." arXiv:2511.05766
- BIRD Annotation Error Rate (2026). "Text-to-SQL Benchmarks Are Broken." CIDR Proceedings. https://vldb.org/cidrdb/2026/text-to-sql-benchmarks-are-broken-an-in-depth-analysis-of-annotation-errors.html
- Text-to-SQL Fundamental Challenges (2025). arXiv:2501.18197. https://arxiv.org/html/2501.18197v1
- ToxicSQL SQL Injection (2025). "Are Your LLM-based Text-to-SQL Models Secure?" arXiv:2503.05445
- Goodhart's Law in Agent Evaluation (2026). PactLabs / Armalo. https://www.agentpact.ai/labs/research/2026-03-17-goodharts-law-agent-evaluation-gaming
- Prompt Versioning Best Practices (2025). https://www.getmaxim.ai/articles/prompt-versioning-and-its-best-practices-2025/
- LLM Evaluation Reproducibility (2025). https://blog.promptlayer.com/why-llm-evaluation-results-arent-reproducible-and-what-to-do-about-it/
- SQLGlot Issues: T-SQL OUTER APPLY (#5874), Join Kind Normalization (#5470), AST Diff (#3581). https://github.com/tobymao/sqlglot
- Structured Output Reliability (2025). Fordel Studios. https://fordelstudios.com/research/structured-outputs-production-systems
- LLM Evaluation 2025 Year in Review. Goodeye Labs. https://www.goodeyelabs.com/insights/llm-evaluation-2025-review
