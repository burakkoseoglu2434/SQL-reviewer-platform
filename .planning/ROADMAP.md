# Roadmap

## Phase 1: Discovery & Golden Dataset Setup (Phase 1)
- Sabit error listesi ve taxonomy'nin cikarilmasi.
- 50 - 100 benchmark golden case sorusu / sql / condition uretimi (JSONL veya DB schema).
- Business rule (Metric Dictionary) mappinglerinin ayarlanmasi.

## Phase 2: MVP Multi-Layer Judge (Phase 2)
- Rule Engine implementasyonu (regex checker, forbidden constraint kontrolleri).
- Basic Sandbox Execution Validator entegrasyonu (parse / satır / sütun check).
- LLM Judge Prompt 1. versiyonunun tasarimi ve entegrasyonu.
- Final JSON formatlama validatorü ve return tipi kararlari.

## Phase 3: Eval Runner Platform (Phase 3)
- Dataset (Golden list) üzerinde toplu koşu yapacak "Batch Runner"ın yazılması.
- Composite Score birlestirmesinin gerceklestirilmesi.
- Pass error, recall run comparison batch metric raporlama.

## Phase 4: Human Review & Feedback Loop (Phase 4)
- İnsanin müdahale edeceği "HUMAN REVIEW" bucketinin ekrana düsürülüp rule revizyonu saglayan döngünün API / Log katmani.

## Phase 5: Online Shadowing (Phase 5)
- Productiondan request alan agent'in result'ını eşzamanlı pipeline'a sürme entegrasyonu.
