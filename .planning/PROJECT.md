# SQL Agent Değerlendirme Platformu

## What This Is
Golden dataset, business rules ve sandbox query execution kullanarak LLM'in ürettiği SQL ve cevapları çok katmanlı biçimde değerlendiren; rule-based kontroller, execution equivalence ve LLM-as-a-judge skorlarıyla eksikleri tespit edip approve / rewrite / human_review kararları üreten bir SQL evaluation ve quality assurance platformu.

## The Problem
Mevcut SQL analizi/üretimi yapan agent'in ürettiği sonuçların doğruluğunun otomatik olarak ve ölçeklenebilir bir biçimde ölçülememesi. Sadece bir LLM string kıyası yeterli olmadığı için farklı hata tiplerini (yanlış tablo kullanımı, date filter eksikliği, vb.) yakalayacak ve insan incelemesi (human-in-the-loop) sistemini de içeren kapsamlı bir eval altyapısına ihtiyaç var.

## Core Value
Agent'in hatalarını production'da değil, test/değerlendirme aşamasında yakalayarak, SQL agent güvenirliğini test edilebilir, kalibre edilebilir, metriklerle takip edilebilir ve otomatik olarak izlenebilir hale getirmek.

## Requirements

### Validated
(None yet — ship to validate)

### Active
- [ ] 3 Girdi mekanizmasının entegrasyonu (Candidate SQL, Gold SQL, Business Rules)
- [ ] Rule Engine Katmanı (deterministik, regex ve parsing bazlı kurallar)
- [ ] Execution Validator Katmanı (sorguları çalıştırma, satır sayısı/şema/sonuçları karşılaştırma)
- [ ] LLM Judge Katmanı (kullanıcı niyetine ve açıklamalara uygunluk)
- [ ] Composite Scoring Logic (%30 rule, %25 exec, %25 semantic, %10 answer grounding, %10 format)
- [ ] Karar mekanizması (Approve, Rewrite, Human Review, Reject)
- [ ] Yapılandırılmış Çıktı Üretimi (Karar, skor, error tipi, yönlendirmeler, evidence JSON scheması)
- [ ] Veritabanı Depolama Modeli (Golden Cases, Judge Runs, Scores, Issues, Human Reviews tabloları)
- [ ] Reporting Dashboard Data Export (pass rate, critical recall vb. metriklerin çıkartılması)

### Out of Scope
- [Tamamen Agent'i kendi kendine yeniden eğiten offline fine-tuning loop] — Şimdilik sadece eval tarafı (evaluation pipeline) geliştiriliyor.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Judge ve Eval sistemlerini ayırmak | Ayrı katmanlar olmalı (Judge SQL'i değerlendirir, Eval Judge'ı değerlendirir) | — Pending |
| 3 Katmanlı Judge | Sadece LLM evaluator yetersiz, string match, schema ve business code execution için ayrı kontroller şart | — Pending |

---
*Last updated: 2026-03-31 after initialization*

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state
