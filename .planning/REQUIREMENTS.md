# Requirements

## 1. Table Stakes

- **Rule Engine Validator**: Deterministik olarak Regex ve parsing uzerinden hata tespiti (ornek: `current_date` kullanimi, missing mandatory filters vs).
- **Execution Validator**: Golden dataset içindeki referans SQL sorgusu ile Agent'ın ürettiği sorgunun sonuç satır sayısını ve yapıları sandbox ortamında kontrol edilecek.
- **LLM Judge**: LLM, "rule" ve "execution" sonuçlarını ve business rules (metric definition) ele alıp niyet kontrolünü yapıp nihai output çıkartacak.
- **Structured Output**: Result (Karar), tek formatlama uzerinden donecek (JSON scheması, reason arrayi, vs).
- **Composite Scoring**: Kurallar %30, Calisma/Exec %25, Semantik %25, vs. agirlikli mantik uzerinden `approve/rewrite/human_review/reject` state'i belirlenecek.
- **Human-in-the-Loop Yönlendirmesi**: Belirli threshold uzerindeki (örn. 0.40 - 0.59) sonuclar manual review sistemine paslanacak.
- **PostgreSQL Log Storage**: Incelemelerin, golden case'lerin db icinde normalize kaydedilmesi (`judge_runs`, `golden_cases`, `judge_scores`, vs.)

## 2. Core Scenarios

- **Offline Benchmark Modeli**: Sabit dataset üzerinden rule iteration ve judge regression ölçümü yapılması.

## 3. Post-MVP (Differentiators)

- **Online Production Shadowing**: Canlı loglar dinlenerek fail case'lerin offline golden dataset tablosuna geri çevrilmesi.
- **Reporting Dashboard**: Error tiplerine veya domainlere göre fail reporting çıkartılması.
