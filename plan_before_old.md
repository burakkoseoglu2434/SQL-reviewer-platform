

Aşağıdaki plan, senin anlattığın use case’e tam oturur:

**Amaç:** DB’den gelen gerçek sorgu/çıktıları alıp, bunları golden dataset ile karşılaştıran; rule-based kontroller + LLM-as-a-judge + opsiyonel execution/result karşılaştırması ile eksikleri işaretleyen bir değerlendirme sistemi kurmak.

Bu yaklaşım, OpenAI’nin SQL eval örneğindeki “JSON şema kontrolü + SQL çalıştırma testi + LLM ile relevancy değerlendirmesi” modelini genişletiyor. OpenAI’nin grader rehberi de string check, text similarity, score model grader ve Python code execution gibi grader türlerini birlikte kullanmayı öneriyor. LangSmith tarafı da bunu pratikte “dataset → evaluators → experiment → sonuç analizi” akışıyla çerçeveliyor; offline eval’i geliştirme aşamasında, online eval’i ise production izlemede kullanmayı öneriyor. ([cookbook.openai.com][1])

---

## 1. Ürünün net kapsamı

Projeyi 3 ayrı katman gibi düşün:

### A. İncelenen sistem

Bu senin SQL agent’ın.
Input: doğal dil soru
Output: SQL, varsa answer text, varsa result set özeti

### B. Judge sistemi

Bu proje.
Görevi:

* SQL doğru mu?
* Golden dataset ile ne kadar uyumlu?
* Hangi hata tipleri var?
* Düzeltme önerisi ne?
* Human review gerekir mi?

### C. Eval platformu

Bu da judge sistemini ölçen katman.
Görevi:

* prompt v1 vs v2 karşılaştırmak
* model A vs model B karşılaştırmak
* rule coverage görmek
* regression yakalamak

En kritik tasarım kararı şu:
**Judge sistemi ile eval platformunu ayır.**
Judge SQL’i puanlar; eval platformu da judge’ın kendisini puanlar.

---

## 2. Başarı kriterleri

Projeye başlamadan önce başarıyı tanımla. Ben şu KPI’larla giderdim:

* **Critical error recall**: kritik hataları kaçırmama oranı
* **False positive rate**: doğru sorguyu gereksiz boSzma oranı
* **Decision accuracy**: approve / rewrite / human_review karar doğruluğu
* **Error-type recall**: beklenen hata tiplerini yakalama oranı
* **Judge explanation quality**: öneri ve açıklamanın uygulanabilirliği
* **SQL-result equivalence rate**: gold query ile aynı sonuca ulaşıp ulaşmama
* **Review latency / cost**: bir case’in kaç saniye ve kaç token tuttuğu

OpenAI’nin SQL eval örneği de özellikle tutarlılık, şema doğruluğu, SQL’in çalışabilirliği ve sorgunun kullanıcı isteğine uygunluğu gibi katmanları ayrı ayrı test ediyor. ([cookbook.openai.com][1])

---

## 3. Dataset stratejisi

### 3.1 Golden dataset şeması

Her case tek kayıt olsun. Minimum alanlar:

```json
{
  "id": "anomaly_001",
  "question": "2026 Ocak ayında anomalide olan satışların toplam cirosu nedir?",
  "candidate_sql": "SELECT ...",
  "gold_sql": "SELECT ...",
  "gold_result_fingerprint": "...",
  "expected_review": {
    "decision": "rewrite",
    "severity": "high",
    "error_types": ["wrong_fact_source", "missing_required_filter"]
  },
  "business_context": {
    "intent": "anomaly",
    "required_filters": ["is_anomaly = 1"],
    "metric_definition": "turnover = sum(net_amount_wovat)",
    "required_grain": "overall"
  },
  "agent_output": {
    "answer_text": "...",
    "table_preview": []
  },
  "metadata": {
    "domain": "anomaly",
    "difficulty": "hard",
    "source": "real_failure"
  }
}
```

### 3.2 Dataset bucket’ları

Dataset’i 4 kovaya böl:

* **Positive**: approve edilmesi gereken temiz query’ler
* **Negative**: net hatalı query’ler
* **Borderline**: tartışmalı ama anlaşılır query’ler
* **Adversarial**: çalışan ama semantik olarak yanlış query’ler

### 3.3 Coverage matrisi

Her case’i şu eksenlerde etiketle:

* domain: anomaly / clearance / stock / ciro / actual price
* error type: join, grain, metric, time, filter, source, answer overclaim
* difficulty: easy / medium / hard
* ambiguity: none / medium / high

Böylece dataset dengesi ölçülebilir olur.

---

## 4. Judge’ın değerlendirme modeli

Ben tek bir judge skoru yerine **çok katmanlı judge** kurarım.

### Katman 1 — Rule-based checks

Deterministic kurallar:

* SQL parse ediliyor mu?
* Forbidden statement var mı?
* Mandatory filter var mı?
* Wrong fact source var mı?
* Join multiplication riski var mı?
* Explicit date istendiği halde CURRENT_DATE kullanılmış mı?
* Grain uyuşmazlığı açık mı?

### Katman 2 — Execution-based checks

Mümkünse:

* candidate SQL’i sandbox DB’de çalıştır
* gold SQL’i çalıştır
* sonuçları normalize edip karşılaştır
* row count, aggregate totals, schema shape, null behavior farklarını hesapla

OpenAI’nin SQL eval notebook’u da SQL validasyonunda gerçek execution testini kullanıyor; CREATE/SELECT ifadelerini SQLite üzerinde çalıştırarak syntactic correctness’i doğruluyor. ([cookbook.openai.com][1])

### Katman 3 — LLM-as-a-judge

Bu katman rule ve execution sinyallerini yorumlar:

* Sorgu kullanıcı niyetini gerçekten karşılıyor mu?
* Hata kritik mi?
* Sonuç aynı olsa bile yöntem business rule’a aykırı mı?
* Answer text, SQL’in desteklediğinden daha ileri bir yorum yapıyor mu?

### Katman 4 — Composite decision

Final karar:

* `approve`
* `rewrite`
* `human_review`
* `reject`

Bu yapı, OpenAI grader rehberindeki “farklı grader türlerini birleştirme” mantığıyla uyumlu; LangSmith tarafı da code evaluators + LLM-as-judge + human review birleşimini destekliyor. ([OpenAI Platform][2])

---

## 5. Judge çıktısı standardı

Judge output’unu mutlaka structured yap:

```json
{
  "decision": "rewrite",
  "overall_score": 0.41,
  "overall_severity": "high",
  "confidence": 0.88,
  "error_types": [
    "wrong_fact_source",
    "missing_required_filter"
  ],
  "scores": {
    "rule_score": 0.20,
    "execution_score": 0.10,
    "semantic_score": 0.55,
    "answer_support_score": 0.30
  },
  "issues": [
    {
      "type": "wrong_fact_source",
      "severity": "high",
      "message": "Sales metric is aggregated from anomaly table instead of fact_sales.",
      "evidence": "Metric dictionary says sales metrics must come from fact_sales."
    }
  ],
  "rewrite_guidance": [
    "Use fact_sales as metric source",
    "Add is_anomaly = 1",
    "Keep date filter on January 2026"
  ]
}
```

Bu sayede:

* dashboard kolay kurulur
* compare-runs yapılır
* rule coverage ölçülür
* human review kolaylaşır

---

## 6. DB’den sorgu alma stratejisi

Senin cümlen önemli: “DB’den sorguları alıp golden dataset ile karşılaştıracak.”

Bunu 3 modda planla:

### Mod 1 — Offline benchmark

DB’den query çekmiyorsun; dataset sabit.
Amaç:

* prompt iteration
* model kıyaslama
* regression

### Mod 2 — Historical trace replay

DB veya log tablosundan geçmiş agent çalışmaları çekilir:

* question
* generated SQL
* final answer
* execution metadata
* runtime errors

Bunlar judge’dan geçirilir, sonra “golden benzeri” bir trace dataset oluşur.

### Mod 3 — Production online evaluation

Canlı sistemden akan query’ler sample edilip judge’a verilir.
LangSmith’in online evaluation modeli tam bu mantığı destekliyor: production trace’ler üzerinde reference-free evaluator çalıştırıp sonra failing örnekleri offline dataset’e geri beslemek. ([LangChain Belgeleri][3])

---

## 7. Mimari

Ben şu servislere bölerdim:

### 7.1 Ingestion service

Kaynaklar:

* golden dataset dosyaları
* DB trace tabloları
* agent run logs
* manual annotations

### 7.2 Normalization service

Normalize eder:

* SQL formatting
* identifier casing
* date literal standardization
* table alias cleanup
* result set canonicalization

### 7.3 Rule engine

Python tabanlı.
Kural aileleri:

* syntax rules
* schema rules
* domain rules
* answer-support rules

### 7.4 Execution validator

Sandbox DB’ye gider.
Yapacakları:

* parse / explain
* safe execute
* result hash
* row stats
* schema shape comparison

### 7.5 LLM Judge service

Input:

* question
* candidate SQL
* gold SQL
* candidate result summary
* gold result summary
* rule findings
* business rules

Output:

* decision
* scores
* issues
* rewrite guidance

### 7.6 Eval orchestrator

Dataset üstünde run açar.
OpenAI eval yaklaşımındaki “unit tests + evaluation + reporting” ve LangSmith’in “dataset → experiment → compare” akışının karşılığı budur. ([cookbook.openai.com][1])

### 7.7 Reporting UI / dashboard

Gösterecekler:

* total pass rate
* critical recall
* by-domain performance
* by-error-type performance
* regressions
* worst 20 cases

---

## 8. Karşılaştırma mantığı

Sadece SQL string karşılaştırma yapma. Bunun yerine 5 ayrı comparison katmanı kullan:

### 8.1 Exact structural checks

* forbidden tokens
* required filters
* required tables
* required grouping
* required order by

### 8.2 AST / shape comparison

* aggregation var mı
* group by aynı grain’de mi
* join sayısı
* subquery kullanımı
* selected metric family

### 8.3 Result equivalence

* same row count?
* same aggregates?
* same sorted top-k?
* tolerance-based numeric equality?

### 8.4 Semantic comparison

LLM judge’a sor:

* candidate SQL user intent’i karşılıyor mu?
* gold query ile aynı business outcome’a ulaşıyor mu?
* candidate daha riskli bir yol izliyor mu?

### 8.5 Answer grounding

* answer text candidate result ile uyumlu mu?
* answer unsupported conclusion içeriyor mu?

Bu son madde çok önemli; çoğu hata SQL’de değil, answer yorumunda olur.

---

## 9. LLM-as-a-judge prompt tasarımı

Judge prompt’unu “genel değerlendirme” değil, rubric bazlı yap.

### System prompt iskeleti

* Rol: senior analytics reviewer
* Amaç: candidate query ve answer’ı gold reference’a göre değerlendirmek
* Öncelikler:

  1. business correctness
  2. metric definition
  3. grain alignment
  4. time filter alignment
  5. source table correctness
  6. unsupported interpretation

### Input blokları

* user question
* business rule summary
* candidate SQL
* gold SQL
* candidate result summary
* gold result summary
* rule findings
* execution findings

### Output

* JSON only
* score 0–1
* decision
* issues[]
* rewrite_guidance[]

OpenAI grader rehberi structured graders ve JSON tabanlı değerlendirme mantığını özellikle destekliyor. ([OpenAI Platform][2])

---

## 10. Scoring sistemi

Ben weighted scoring öneririm:

* `rule_score` = %30
* `execution_equivalence_score` = %25
* `semantic_judge_score` = %25
* `answer_grounding_score` = %10
* `format/schema_score` = %10

### Örnek

* Candidate SQL çalışıyor ama wrong_fact_source var → rule düşük, semantic düşük
* Candidate SQL gold ile aynı sonucu veriyor ama answer overclaim yapıyor → execution yüksek, answer düşük
* Candidate SQL gold’dan farklı ama eşdeğer → execution yüksek, semantic yüksek, exact structure düşük ama overall approve olabilir

### Karar eşiği

* `>= 0.85` → approve
* `0.60–0.84` → rewrite
* `0.40–0.59` → human_review
* `< 0.40` → reject

Bu eşikleri sonra human annotation ile kalibre et.

---

## 11. Human-in-the-loop

Judge her şeyi otomatik çözemeyecek. Bu yüzden insan onayı için ayrı akış tasarla.

### Human review’e düşecek case’ler

* ambiguous metric definition
* ambiguous business term
* contradictory rules
* gold ve candidate aynı sonucu veriyor ama farklı mantıkta
* judge confidence düşük

### İnsan ekranında gösterilecekler

* question
* candidate SQL
* gold SQL
* result diff
* judge issues
* suggested decision

### Human annotation alanları

* final label
* corrected error types
* corrected severity
* comment
* reusable rule suggestion

Bu anotasyonlar sonra judge kalibrasyon datası olur.

---

## 12. Teknik stack önerisi

### Backend

* Python
* FastAPI
* SQLAlchemy / psycopg
* Pydantic

### Eval orchestration

* Basit Python runner ile başla
* İstersen LangSmith’e trace ve experiment katmanı ekle
  LangSmith’in `evaluate()` yapısı dataset, evaluator ve experiment mantığını direkt destekliyor. ([langsmith-sdk.readthedocs.io][4])

### Storage

* Postgres:

  * `golden_cases`
  * `judge_runs`
  * `judge_scores`
  * `human_reviews`
  * `rule_findings`
* Object storage:

  * raw result snapshots
  * diff reports

### UI

* İlk aşamada notebook + CSV + JSON
* Sonra Streamlit veya internal dashboard

---

## 13. DB şema önerisi

### `golden_cases`

* id
* question
* candidate_sql
* gold_sql
* expected_decision
* expected_error_types
* business_context_json
* metadata_json

### `judge_runs`

* run_id
* case_id
* model_version
* prompt_version
* rules_version
* timestamp
* latency_ms
* token_usage

### `judge_scores`

* run_id
* overall_score
* rule_score
* execution_score
* semantic_score
* answer_score
* decision
* confidence

### `judge_issues`

* run_id
* issue_type
* severity
* message
* evidence

### `human_reviews`

* run_id
* reviewer_id
* final_decision
* corrected_error_types
* notes

---

## 14. Geliştirme fazları

### Faz 0 — Discovery

1 hafta

* use case sınırlarını netleştir
* error taxonomy çıkar
* 50–100 golden case topla
* metric dictionary yaz

### Faz 1 — MVP judge

2 hafta

* rule engine
* basic execution validator
* ilk LLM judge prompt
* JSON output schema

### Faz 2 — Eval platform

1–2 hafta

* batch runner
* scoring
* reports
* run comparison

### Faz 3 — Human review ve kalibrasyon

1 hafta

* annotation UI
* confidence threshold
* rule tuning

### Faz 4 — Production shadow mode

1–2 hafta

* canlı trace’leri al
* online judge koş
* failure → offline dataset geri besle

Bu döngü, LangSmith’in önerdiği offline + online sürekli iyileştirme yaklaşımıyla uyumlu. ([LangChain Belgeleri][3])

---

## 15. Riskler

### A. Gold SQL tek doğruymuş gibi davranmak

Çözüm:

* result equivalence ve semantic equivalence ekle

### B. Judge’ın overfit olması

Çözüm:

* borderline ve adversarial bucket tut
* compare-runs ile regression izle

### C. Rule explosion

Çözüm:

* önce recurring high-impact kurallar
* nadir edge-case’leri human review’e bırak

### D. Judge’ın kendisinin halüsinasyon yapması

Çözüm:

* rule + execution sinyali olmadan sadece LLM’e güvenme
* structured output kullan
* confidence ve evidence zorunlu olsun

### E. Production DB’ye zarar verme

Çözüm:

* sandbox / read replica
* query timeout
* row limit
* read-only credentials

---

## 16. İlk sprint backlog

İlk sprintte şunları hedeflerdim:

1. Error taxonomy’yi sabitle
2. 50 case’lik golden dataset’i JSONL veya Postgres’e yükle
3. Rule engine v1 yaz
4. Candidate vs gold SQL execution sandbox’ını kur
5. Judge prompt v1 yaz
6. Judge JSON schema’yı sabitle
7. Batch eval runner yaz
8. İlk raporu üret:

   * pass rate
   * critical recall
   * top 10 failed cases

---

## 17. En pratik başlangıç mimarisi

Başlangıç için en verimli yol şu:

**Rule engine + sandbox execution + single LLM judge + batch evaluator**

Yani hemen büyük platform kurma. Şöyle başla:

* Python service
* Postgres’te golden case tablosu
* nightly batch eval
* CSV/JSON rapor
* haftalık compare-runs

Sonra şunları eklersin:

* answer judge
* online eval
* annotation UI
* experiment dashboard

---

## 18. Sana önerdiğim nihai proje cümlesi

Bunu proje tanımı olarak kullanabilirsin:

**“Golden dataset, business rules ve sandbox query execution kullanarak LLM’in ürettiği SQL ve cevapları çok katmanlı biçimde değerlendiren; rule-based kontroller, execution equivalence ve LLM-as-a-judge skorlarıyla eksikleri tespit edip approve / rewrite / human_review kararları üreten bir SQL evaluation ve quality assurance platformu.”**

İstersen bir sonraki adımda bunu daha da somutlaştırıp sana:

* **DB schema DDL**
* **judge output Pydantic modeli**
* **LLM judge prompt v1**
* **ilk eval runner kod skeleton’ı**
  çıkarayım.

[1]: https://cookbook.openai.com/examples/evaluation/how_to_evaluate_llms_for_sql_generation "How to evaluate LLMs for SQL generation"
[2]: https://platform.openai.com/docs/guides/graders/ "Graders | OpenAI API"
[3]: https://docs.langchain.com/langsmith/evaluation?utm_source=chatgpt.com "LangSmith Evaluation - Docs by LangChain"
[4]: https://langsmith-sdk.readthedocs.io/en/stable/evaluation/langsmith.evaluation._runner.evaluate.html?utm_source=chatgpt.com "evaluate — 🦜️🛠️ LangSmith documentation"
