# SQL Değerlendirme & LLM Hakem Platformu (Ayar Şeması İncelemesi)

Bu belge, projenin baştan sona nasıl çalışacağını ve verilerin nasıl bir tünelden (pipeline) geçeceğini gösteren **Draw.io mimari şemasının** herkes tarafından kolayca anlaşılabilecek adım adım detaylı anlatımıdır. 

Sistem toplamda birbirine veri taşıyan 5 geliştirme fazından (Phase) oluşmaktadır. 

---

## 🗄️ Faz 0: Altın Veri Setini Hazırlama (Golden Dataset Foundation)
Uygulama kodunun çalışacağı en temel zemindir. Şemanın en başındaki başlangıç noktasıdır. Kodlanacak tüm "Test sisteminin" sağlıklı çalışabilmesi için ilk kontrol burasıdır.
1. **Raw Dataset (Ham Veri):** Elimizde karmaşık haldeki işlenmemiş SQL veri setidir.
2. **Validate Schema & Buckets (Doğrulama ve Sınıflandırma):** Bu ham veriler önceden belirlenmiş şablonlara göre düzenlenir. Eksik kolonlar tamamlanır, sorular "zorluk" veya "beklentisine" göre sepetlere (buckets) ayrılır.
3. **Golden Dataset Store (Altın Veri Havuzu):** Hatalarından tamamen arındırılır ve testte kullanmak için izole veritabanlarına kalıcı olarak kaydedilir.
*👉 Görevi:* Sonradan işlem yapacak tüm karar sistemine, kesin güvenilir ve şemasız olmayan bir "Test Kaynağı" sunmaktır.

---

## 🚦 Faz 3: Toplu Başlatıcı Kütlesi (Batch Runner Init)
3. Fazın bir parçası olmasına rağmen akışta sistem buradan başlatılır (Trigger olur).
1. **Batch Eval Run:** Değerlendirme sisteminin başlama komutudur. 
2. **Load 968 Golden Cases:** Faz 0'da elde ettiğimiz 968 harika, altın kelimelerle bezenmiş SQL tablosu test edilmek üzere RAM'e/Hafızaya çekilir. 
3. **Parallel Async Runner:** İşlemi asenkron (eşzamanlı) çalıştırarak donanımın kökünü kullanır. Tüm sorguları, hızla doğrulanmak üzere ana tünele "Çoklu parçalar halinde" ateşler. (Spawns Threads).

---

## ⚙️ Faz 1: Hatasız Test Hattı Algoritmaları (Deterministic Pipeline)
Yapay Zekanın veya tahmin algoritmalarının devreye girmediği katı mantık katmanıdır. Gelen SQL, çok ciddi bir 4'lü filtreden geçirilir. Hata oranının en az %70'i bu basit algoritmik katmanda elenir ki yapay zeka API maliyetinden tasarruf edilsin!
1. **Single Candidate SQL:** Asıl incelenecek SQL sorgusu tünele girer.
2. **Rule Engine (Kural Motoru):** En ilkel filtredir. SQL'in yapısı string (yazı) kuralları gereği kabul edilebilir mi diye basit bir 'If/Else' kontrolü yapar. Hatalıysa hemen **(Early Reject)** klasörüne yani direkt elenmeye düşer.
3. **Execution Validator - DuckDB:** Kuralları geçen SQL, güveli bir "Kum havuzunda" (DuckDB içinde) fiziksel olarak çalıştırılır. Dosyayı mı patlattı? Kod çalışmadı mı? Sorgu hatalı mı? Varsa Early Reject'e (reddedilmeye) gider. 
4. **AST Validator:** Gerçekten çalışan SQL kodunun ağacı çıkarılır ve orijinal altın cevap ile matematik kurallarına göre bir nevi "kardeşler mi" ("hash") karşılaştırması yapılır.
5. **Result Comparator:** Test cevabı veriler ile uyumlu oranda bir değer mi çıkarmış ona bakılır. (Match Ratio).
*👉 Çıktı:* Tüm süzgeçleri geçen veya Early Reject olan her bir satır işlem için "EvalContext" isminde çok dökümante edilmiş kapsamlı bir analiz paketi (dosyası/objesi) elde edilir. 

---

## 🧠 Faz 2: Yapay Zeka Hakemliği ve Puanlama (LLM Judge & Scoring)
Mantıksal eleklerden başarıyla geçen "EvalContext", sistemin makine algoritmalarıyla test edilemeyecek kadar derin ve anlamsal detayların denetimi için Yapay Zeka Hakemine aktarılır. Bu aşama rastgele promptlarla çalışmaz, aksine arkasında çok katı istatistiksel ve matematiksel kurallar yatar.

1. **LLM Judge (Yapay Zeka Hakemi):** Gelen SQL kodunu tam olarak şu 3 kategoride sınava tabi tutar ve 0.0 ile 1.0 arası puanlandırır:
   - *A) Niyet Sadakati (Intent Fidelity):* Metrikler, zaman aralığı ve asıl istenilen (Örn: "Aylık kazanç") gerçekten karşılanmış mı?
   - *B) İş Mantığı Doğruluğu (Business Correctness):* Doğru tablolara inilmiş mi ve kurumun zorunlu kıldığı (Örn: `is_active=1` olmalı) kurallar SQL kodunda işlenmiş mi?
   - *C) Köklerine Bağlılık (Answer Grounding):* Doğal dil ile üretilen cevap, gerçekten alttaki SQL sorgusundan çıkan sayı veya isimle ispat edilebiliyor mu? (Uygulamanın halüsinasyon-overclaiming kontrolü)

2. **Hakem Çıktısı (Kapsamlı JSON Nesnesi):** Değerlendirme sonucunda LLM düz yazı değil sadece yazılımcı sistemini besleyecek formatlı bir JSON döner. Bu nesnede verilen puanların yanında, *Gerekçe (Rationale)*, hatanın veya doğrunun *Kanıtı (Cited Evidence)* ve geliştirici sistem için *Yeniden Yazım Tavsiyeleri (Rewrite Guidance)* bulunur.

3. **Composite Scoring Engine (Bileşik Puanlama Matematiği):** LLM'in anlamsal (semantic) skorları ile (1. Faz)'dan elde edilen algoritmik test skorları tek bir potada ağırlıklı olarak eritilir. Şema formülü şöyledir:
   - *Final Skoru (0.0-1.0)* = Niyet(%25) + Başarılı Çalışma Durumu(%20) + Semantik/AST(%15) + İş Mantığı(%15) + Sonuç Eşleşmesi(%10) + Köklerine Bağlılık(%10) + Risk Oranı(%5)

4. **İstisnasız Red Kuralları ve Nihai Karar (Override Floor Rules):** 
   - Eğer SQL sorgusu önceki adımlardaki DuckDB testlerinde hata verdiyse VEYA *Niyet Sadakati (Intent Fidelity)* %20'nin (0.20) altındaysa; diğer puanları tam (%100) bile çıksa anında istisnasız bir şekilde **🔴 Reddedildi (Reject)** kararı çıkarılır. Kısmen doğru bir çöp koda şans verilmez.
   - Bu devasa kırmızı çizgi kurallarını geçen SQL'ler ancak şu sınırlarla atanır: **🟢 ONAY** (>= 0.85), **🟡 YENİDEN YAZILMALI** (0.65 - 0.84), **🟠 İNSAN İNCELEMESİ** (0.40 - 0.64) veya **🔴 REDDEDİLDİ** (< 0.40) 

5. **JudgeQualityScorer:** Peki Yapay zekanın sağladığı bu değerlendirme ne kadar güvenilir? Yapay Zekanın uydurma üretip üretmediği (ikinci kez fazladan LLM masrafına girilmeden) istatistiksel yollarla incelenir.

---

## 📊 Faz 3 (Devamı): Raporlama ve İstatistikler (Reporting)
Sonuca ulaşan tüm sistem çıktıları "Terminal'e (Ekrana)" dökülmeden önce derlenir.
1. **Batch Aggregator (Sonuç Toplayıcı):** Tek tek koşan binlerce test bir havuza alınır ve toparlanır.
2. **Hallucination & Judge Reliability:** Algoritma analizleri tamamlar; LLM nerede saptı?, "LLM'lerin bizim Orijinal Test ile ne kadar oranda uyumlu gittiği" (False Positive oranı) ve hakem kalitesi ölçülür.
3. **Report Generator & Terminal Report:** Ve günün en güzel anı! Program ekranına çok detaylı tablo çizilir. Pass_Rate, Uyumsuzluk, Maliyet (Tokenler), Ortalama test süresi vs vs. tüm rapor görsel zerafetle sunulur.

---

## 💾 Faz 4: İzlenebilirlik ve Endüstri Depolaması (Observability & Storage Layer)
Uygulamanın çalışıp işe yaradıktan sonra şirket standartına kazandırılması gereken, kurumsal arşiv fazıdır.
1. **Observability Metadata:** Tüm metrikler, bağlantıların milisaniyelik gecikmeleri gibi sistem telemetreleri paket haline getirilir.
2. **ResultStore Protocol:** OOP esnekliği devreye girer. Depolama altyapısı "JSON olarak File Storage"ye (yerel disk depolama) mi?, yoksa asıl büyük kurumsal veritabanı yığını olan "PostgreSQL" sunucularına mı kaydedileceğini bir protokolle denetler ve istenilene tüm rapor içeriğini SQL insert verisi olarak kaydeder. 

