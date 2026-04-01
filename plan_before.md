# SQL Evaluation & LLM-as-a-Judge Platform (Updated Plan)

## Amaç

Golden dataset'i aylık olarak ürün ve business değişikliklerine göre gözden geçirip güncel agent'a replay eden; SQL, sonuç ve answer'ı execution-based, semantic, rule-based ve LLM-as-a-judge katmanlarıyla değerlendiren; ayrıca latency, token kullanımı, kullanılan model ve hallucination rate gibi metrikleri izleyen bir **SQL evaluation, regression ve governance platformu** kurmak.

---

# 1. Sistem Bileşenleri

## 1.1 SQL Agent (Mevcut Sistem)

* Input: Natural language question
* Output:
  * SQL query
  * Answer text
  * Result preview / aggregation

---

## 1.2 Judge Sistemi

Görevleri:

* SQL doğruluğunu değerlendirmek (execution + semantic + structural)
* Business correctness kontrolü yapmak
* Answer grounding kontrolü yapmak
* Risk analizi yapmak
* Eksikleri ve hataları tespit etmek
* Rewrite guidance üretmek
* Kendi çıktısının kalitesini izlemek (judge output quality)

---

## 1.3 Evaluation Platform

Görevleri:

* Golden dataset üzerinden batch evaluation
* Model / prompt / rule karşılaştırması
* Regression tespiti
* Monthly replay
* Performans ve maliyet analizi
* BIRD benchmark uyumluluk testi (opsiyonel)

---

# 2. LLM-as-a-Judge Kriterleri (Genişletilmiş)

Judge aşağıdaki **8 ana boyutta** değerlendirme yapar.

---

## 2.0 Evaluation Sequence (Judge'ın İzlediği Sıra)

Her case için judge bu sırayı takip eder:

```
1. Query, kullanıcı isteğini karşılıyor mu?        → Intent Fidelity
2. SQL yapısal olarak geçerli ve parse edilebilir?  → Structural Validity
3. SQL hata vermeden çalışıyor mu?                 → Execution Accuracy
4. Döndürülen sonuç semantik olarak doğru mu?       → Result Equivalence
5. Business kurallarına uyuyor mu?                  → Business Correctness
6. Answer SQL sonucu tarafından destekleniyor mu?   → Answer Grounding
7. Risk kalıpları var mı?                           → Risk & Reliability
8. Judge açıklamasının kalitesi yeterli mi?         → Judge Output Quality
```

> **Temel ilke (EvaluateGPT'den):** "Fidelity to User Request is Paramount." Kullanıcı isteğine uygunluk her zaman birinci önceliktir.

---

## 2.1 Intent Fidelity

* Query gerçekten soruyu cevaplıyor mu?
* Metric, grain, entity, zaman uyumu var mı?
* Kullanıcının açıkça belirttiği tüm koşullar karşılanmış mı?
* Literal interpretation: kullanıcının kastettiği değil, yazdığı esas alınır

---

## 2.2 SQL Correctness & Execution

### 2.2.1 Structural Validity

* SQL parse edilebilir mi? (valid_sql_rate)
* Syntax hataları var mı?
* AST üretilip üretilemediği kontrol edilir

### 2.2.2 Execution Accuracy

| Metrik | Açıklama |
|--------|----------|
| `execution_accuracy` | SQL hata vermeden çalışıyor mu? |
| `non_empty_execution_accuracy` | Her iki SQL de veri döndürüyor ve sonuçlar eşleşiyor mu? |
| `subset_non_empty_execution_accuracy` | Üretilen SQL sonucu, beklenen sonucun alt kümesiyle eşleşiyor mu? |
| `gold_sql_runtime` | Gold SQL'in çalışma süresi (ms) |
| `predicted_sql_runtime` | Üretilen SQL'in çalışma süresi (ms) |
| `routing_accuracy` | Doğru tablo/schema seçildi mi? |

> **Not:** `non_empty_execution_accuracy` birincil skor metriği olarak kullanılır (IBM/unitxt yaklaşımı).

### 2.2.3 SQL Logic

* Join doğru mu?
* Aggregation doğru mu?
* Filter doğru mu?
* Subquery / CTE mantığı tutarlı mı?

---

## 2.3 Business Correctness

* Doğru fact table kullanılmış mı?
* Zorunlu filtreler eklenmiş mi?
* Metric doğru tanımlanmış mı?
* Grain doğru mu?

**Default behavior kuralları (product-specific):**

* Zaman dilimi belirtilmezse 3/5/10 yıllık CAGR hesaplanır
* "Son dönem" analizi için 5 yıllık lookback varsayılandır
* "Temel güçlü" tanımı için rating ≥ 3.5 eşiği kullanılır
* "Artan" ifadesi ardışık büyüme anlamına gelir (negatif olsa bile)

---

## 2.4 Result Equivalence (Detaylı)

Candidate SQL ile Gold SQL arasındaki sonuç karşılaştırması:

| Kriter | Detay |
|--------|-------|
| **Row Count Match** | Satır sayısı eşleşmeli |
| **Column Count** | Sütun sayısı aynı olmalı |
| **Column Name Match** | Sütun adları case-insensitive eşleşmeli |
| **Data Type Tolerance** | Sayısal değerlerde tolerans: `atol=1e-5` |
| **Row Order Independence** | Sıralama farklı olsa da sonuç doğru sayılır (opsiyonel strict mode) |
| **Value Equality** | Tolerans dahilinde değer eşleşmesi |
| **Subset Matching** | Kısmi eşleşme de puanlandırılır |

> **Semantic Equivalence:** AST normalizasyonu ile syntaktik olarak farklı ama semantik olarak eşdeğer SQL'ler doğru kabul edilir.

---

## 2.5 Answer Grounding

* Answer SQL sonucu tarafından destekleniyor mu?
* Overclaim var mı?
* Unsupported conclusion var mı?
* SQL `TOP 50` döndürürken answer "tüm ürünler" diyorsa → hallucination

---

## 2.6 Risk & Reliability

* LIMIT kullanımı riskli mi?
* CURRENT_DATE / GETDATE() kullanımı var mı?
* Sampling bias var mı?
* JOIN multiplication riski var mı?
* Implicit type cast var mı?
* NULL handling eksik mi?

---

## 2.7 Product Drift & Semantics

* Product vs option ayrımı doğru mu?
* Category/hierarchy değişmiş mi?
* Clearance tanımı güncel mi?
* Deprecated alan kullanılmış mı?

---

## 2.8 Judge Output Quality (Yeni Katman)

Judge'ın kendi çıktısının kalitesi ölçülür. Bu meta-evaluation katmanı judge'ın güvenilirliğini izler:

| Boyut | Ağırlık | Açıklama |
|-------|---------|----------|
| `specificity` | 35% | Judge açıklamaları yeterince spesifik ve somut mu? |
| `completeness` | 25% | Tüm sorunlar tespit edildi mi, eksik issue var mı? |
| `rationale_density` | 20% | Her karar gerekçelendirilmiş mi? |
| `actionability` | 20% | Rewrite guidance uygulanabilir mi? |

> Bu boyutlar ayrıca `judge_quality_score` olarak kaydedilir ve judge'ın zaman içindeki kalibrasyonu izlenir.

---

# 3. Golden Dataset Yapısı

## 3.1 Case Şeması

```json
{
  "id": "case_001",
  "question": "...",
  "gold_sql": "...",
  "gold_answer_text": "...",

  "execution_metadata": {
    "gold_execution_status": "success | empty | error",
    "gold_row_count": 42,
    "gold_column_names": ["date", "revenue", "product"],
    "gold_execution_time_ms": 230,
    "gold_result_fingerprint": "sha256:..."
  },

  "expected_review": {
    "decision": "approve | rewrite | human_review | reject",
    "severity": "critical | high | medium | low",
    "error_types": []
  },

  "business_context": {
    "intent": "...",
    "required_grain": "...",
    "metric_definition": "...",
    "default_behaviors": {},
    "product_semantics_version": "2026-04"
  },

  "metadata": {
    "domain": "...",
    "difficulty": "easy | medium | hard | adversarial",
    "bucket": "positive | negative | borderline | adversarial",
    "bird_benchmark_id": "...",
    "last_reviewed_at": "..."
  }
}
```

---

## 3.2 Dataset Bucket'ları

* **Positive** — doğru, approve edilmesi gereken case'ler
* **Negative** — açık hata içeren, rewrite gereken case'ler
* **Borderline** — insan değerlendirmesi gereken edge case'ler
* **Adversarial** — judge'ı zorlamak için tasarlanmış semantik tuzaklar

---

# 4. Monthly Evaluation Flow

## Her ay yapılacaklar:

### 1. Product & Business Review

* Metric dictionary güncellenir
* Product hierarchy değişimleri incelenir
* Clearance / anomaly tanımları kontrol edilir
* Default behavior kuralları gözden geçirilir

---

### 2. Dataset Review

* Eski / deprecated case'ler kaldırılır
* Ambiguous case'ler düzeltilir
* Yeni edge-case ve adversarial case eklenir
* `product_semantics_version` güncellenir

---

### 3. Agent Replay

Golden dataset'teki tüm sorular tekrar çalıştırılır:

* SQL üretilir
* Answer üretilir
* Result alınır
* Execution metadata kaydedilir

---

### 4. Evaluation

* Gold ile karşılaştırılır (execution + AST + result-set)
* Rule engine çalıştırılır
* LLM Judge çalıştırılır
* Judge output quality skoru hesaplanır
* Composite score hesaplanır

---

### 5. Raporlama

Aşağıdaki metrikler üretilir:

* pass_rate
* critical_error_recall
* hallucination_rate
* non_empty_execution_accuracy
* valid_sql_rate
* routing_accuracy
* avg_latency / p95_latency
* token_usage / token_cost
* model_distribution
* regression_list
* judge_quality_score_avg

---

### 6. Failure Analysis

Fail eden case'ler kategorize edilir:

* model_regression
* rule_eksikliği
* product_drift
* hallucination (answer veya judge)
* execution_error
* routing_error

---

# 5. Observability

Her run'da şu bilgiler tutulur:

```json
{
  "model_name": "...",
  "latency_ms": 1200,
  "input_tokens": 450,
  "output_tokens": 180,
  "total_tokens": 630,
  "prompt_version": "...",
  "rules_version": "...",
  "execution_time_ms": 230,
  "execution_status": "success | empty | error",
  "result_row_count": 42,
  "result_columns": ["date", "revenue"],
  "sql_ast_hash": "sha256:..."
}
```

## İzlenecek Metrikler

* avg_latency / p95_latency
* avg_input_tokens / avg_output_tokens
* total_token_cost
* model_usage_distribution
* execution_success_rate
* empty_result_rate
* avg_result_row_count

---

# 6. Hallucination Tracking

## Tanım

Hallucination = SQL/result tarafından desteklenmeyen iddia

## Türler

### Answer Hallucination
* SQL `TOP 50` döndürür → Answer "genel dağılım" der
* SQL belirli tarih aralığı kısıtlar → Answer "tüm zamanlar" der

### Judge Hallucination
* Yanlış error type üretme
* Var olmayan sütunu hata olarak işaretleme
* Doğru SQL'i yanlış gerekçeyle rewrite etme

## Metrik

```text
hallucination_rate = hallucinated_cases / total_cases
judge_hallucination_rate = judge_errors / total_judge_runs
```

---

# 7. Mimari

```text
Ingestion
   ↓
Normalization
   ↓
Rule Engine
   ↓
Execution Validator
   ↓
AST / Semantic Validator
   ↓
Result Set Comparator
   ↓
LLM Judge
   ↓
Judge Quality Scorer
   ↓
Aggregator
   ↓
Reporting
```

**Katmanların sorumluluğu:**

| Katman | Sorumluluk |
|--------|------------|
| Rule Engine | Structural rules, business rules, risk patterns |
| Execution Validator | SQL çalışıyor mu, boş sonuç mu, hata mı? |
| AST/Semantic Validator | Farklı syntax, aynı anlam mı? |
| Result Set Comparator | Row/column/value düzeyinde karşılaştırma |
| LLM Judge | Bütünsel değerlendirme, rewrite guidance |
| Judge Quality Scorer | Judge çıktısının meta-değerlendirmesi |

---

# 8. DB Şema

## golden_cases

* id
* question
* gold_sql
* gold_execution_status
* gold_row_count
* gold_column_names
* gold_result_fingerprint
* gold_execution_time_ms
* business_context (JSON)
* metadata (JSON)

---

## agent_replay_runs

* case_id
* run_id
* sql
* answer
* model_name
* prompt_version
* latency_ms
* input_tokens / output_tokens
* execution_status
* execution_time_ms
* result_row_count
* result_columns
* sql_ast_hash

---

## judge_scores

* run_id
* overall_score
* intent_fidelity_score
* execution_score
* semantic_score
* result_equivalence_score
* business_correctness_score
* answer_grounding_score
* risk_score
* judge_quality_score
* decision
* confidence

---

## judge_issues

* run_id
* type
* severity
* message
* evidence

---

## hallucination_events

* run_id
* type (answer | judge)
* severity
* evidence

---

## monthly_reports

* report_date
* pass_rate
* non_empty_execution_accuracy
* valid_sql_rate
* routing_accuracy
* hallucination_rate
* judge_hallucination_rate
* avg_latency
* p95_latency
* token_usage
* token_cost
* model_distribution

---

# 9. Scoring Model

## 9.1 Ağırlıklar

| Metrik | Ağırlık | Kaynak |
|--------|---------|--------|
| `intent_fidelity` | 25% | Kullanıcı isteğine uygunluk (1. öncelik) |
| `execution_score` | 20% | non_empty_execution_accuracy |
| `semantic_score` | 15% | AST-based equivalence |
| `business_correctness` | 15% | Rule engine (tablo, filtre, metrik tanımı) |
| `result_equivalence` | 10% | Row/column/value seviyesinde eşleşme |
| `answer_grounding` | 10% | Answer'ın SQL sonucu tarafından desteklenmesi |
| `risk_score` | 5% | Risk kalıpları (LIMIT, CURRENT_DATE, vb.) |

> **Not:** `observability` metrikleri kalite skoruna dahil edilmez; ayrı operasyonel dashboard'da izlenir. `structural_validity` (valid_sql_rate) `execution_score`'un ön koşuludur; 0 ise tüm skor 0'a düşer.

---

## 9.2 Skor Tablosu

| Skor Aralığı | Karar | Açıklama |
|-------------|-------|----------|
| 0.85 – 1.00 | `approve` | Mükemmel: doğru çalışır, doğru sonuç, verimli |
| 0.65 – 0.84 | `rewrite` | Doğru ama verimsiz veya minor eksik |
| 0.40 – 0.64 | `human_review` | Kısmi doğruluk, eksik mantık veya belirsiz intent |
| 0.20 – 0.39 | `reject` | Ciddi hatalar, yanlış yaklaşım, büyük boşluklar |
| 0.00 – 0.19 | `reject` | Temel düzeyde hatalı veya execute edilemiyor |

---

## 9.3 Kritik Override Kuralları

Aşağıdaki durumlardan biri oluşursa overall skor ne olursa olsun karar **`reject`** olur:

* `execution_status = error` (SQL execute edilemiyor)
* `valid_sql_rate = 0` (SQL parse edilemiyor)
* `routing_accuracy = 0` (tamamen yanlış tablo seçimi)
* Kritik hallucination tespit edildi

---

# 10. Development Phases

## Faz 0 — Discovery

* Error taxonomy oluşturma
* Golden dataset oluşturma ve bucket'lama
* Product rules ve default behavior dokümantasyonu
* BIRD benchmark uyumluluk analizi

---

## Faz 1 — MVP Judge

* Rule engine implementasyonu
* Execution validator (SQL çalıştırma + sonuç karşılaştırma)
* AST/Semantic validator
* Result set comparator (row/column/value seviyesi)
* Temel LLM judge (scoring + decision)

---

## Faz 2 — Eval Platform

* Batch runner
* Scoring aggregator
* Regression detection
* Raporlama dashboard'u

---

## Faz 3 — Observability

* Latency / token tracking
* Hallucination tracking (answer + judge)
* Judge output quality scorer
* Model/prompt comparison

---

## Faz 4 — Monthly Governance

* Dataset refresh workflow
* Product drift detection
* Adversarial case generation
* BIRD benchmark entegrasyonu

---

## Faz 5 — Production

* Online eval (shadow mode)
* Real-time judge feedback
* Auto-rewrite suggestion pipeline

---

# 11. Riskler

| Risk | Etki | Önlem |
|------|------|-------|
| Gold SQL tek doğru değil | False negative | Result equivalence + AST normalization ekle |
| Judge hallucination | Güven kaybı | Structured output + evidence requirement; judge_quality_score |
| Product drift | Stale evaluations | Monthly product review + semantics_version |
| Overfitting to golden set | Blind spots | Adversarial dataset + BIRD benchmark |
| Empty result false positive | Yanlış approval | `non_empty_execution_accuracy` primary metric olarak kullan |
| Numeric tolerance | Yanlış rejection | `atol=1e-5` tolerans uygula |

---

# 12. Final Tanım

**Golden dataset'i aylık olarak ürün ve business değişikliklerine göre güncelleyip agent replay ile test eden; SQL doğruluğunu execution-based (non_empty_execution_accuracy), AST/semantic, result-set ve rule-based katmanlarla değerlendiren; LLM-as-a-judge ile bütünsel karar üreten; ayrıca latency, token, model ve hallucination metriklerini izleyerek sistem kalitesini sürekli ölçen, judge'ın kendi output kalitesini de takip eden bir evaluation ve governance platformu.**
