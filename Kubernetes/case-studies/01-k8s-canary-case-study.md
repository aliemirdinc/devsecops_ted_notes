# Case Study: Canary Deploy — İzleme ve Geri Alma

Senaryo:
- Yeni bir sürüm (v2) üretime alınmadan önce küçük bir kullanıcı yüzdesine verilecek.
- Amaç: hata/performans regresyonunu erken yakalamak.

Adımlar:
1. `ratings-v1` (9 replika) ve `ratings-v2` (1 replika) deploy edin.
2. Monitor edin: hata oranı, latency, işlevsellik testleri.
3. Eğer sorun yoksa `ratings-v2` replika sayısını arttırıp `v1`'i azaltın.
4. Problem bulunursa hemen rollback (`kubectl rollout undo`) veya `v2`'yi scale down yapın.

Olası hata senaryoları:
- `v2` memory leak yapıyor → pod OOM → servis hata oranı artar.
- Yeni endpoint auth farkı nedeniyle 500 hataları.

Savunma/Önlemler:
- Canary için hazır metrikler (error rate, p95 latency) + otomasyon (Argo Rollouts veya service mesh ile otomatik rollback).
- Sağlık kontrolleri (readiness probe) doğru ayarlı olmalı.
- Trafik izleme ve logların sürüm bazlı etiketlenmesi.

Öğrenilen dersler:
- Manuel canary basit ve etkili ama dikkatli metrik takibi gerektirir.
- Otomasyon (Argo/Flagger/Service Mesh) riskleri azaltır.
