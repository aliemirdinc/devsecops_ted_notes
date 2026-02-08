# Case Study: CI/CD Image Supply Chain — Malicious Dependency

Senaryo:
- CI pipeline otomatik olarak üçüncü taraf paketleri indiriyor.
- Tedarik zincirinde bir paket ele geçirildi; derlenen image içine kötü amaçlı kod enjekte edildi.

Saldırı zinciri:
1. Geliştirici `requirements.txt` güncellemesi yaptı.
2. CI pipeline paketleri indirdi ve image oluşturdu.
3. Image, registry'ye push edildi; üretime otomatik deploylandı.
4. Kötü amaçlı kod tetiklendi ve veri dışarı aktarıldı.

Savunma & Önlemler:
- Dependency pinning ve veri bütünlüğü (hash) doğrulama.
- SBOM oluşturup CI'da değişiklikleri takip edin.
- Supply-chain signing (Sigstore/Cosign) kullanın: imajları ve artefact'ları imzalayın.
- CI ortamında paket kaynaklarını kısıtlayın ve allowlist kullanın.
- Runtime davranış analizi ve EDR ile anormallikleri tespit edin.

Taktikler:
- Policy: CI push öncesi `trivy` ve SBOM uyumluluk kontrolleri.
- Incident response: registry'den etkilenen image'leri çekip rotasyon, deploy rollback.
