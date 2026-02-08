# Case Study: Secret Leakage in Cluster

Senaryo:
- Uygulama loglarında hassas bilgi (API key) yazdırılıyor.
- Log toplama pipeline'ı (ELK/Fluentd) merkezi sisteme gönderiyor; anahtar burada indekslendi.

Saldırı zinciri:
1. Kötü amaçlı kişi log indekslerini araştırır ve anahtarı bulur.
2. Anahtar ile diğer servislerde yetkisiz işlem yapar.

Nedenleri:
- Uygulama log seviyesinin yanlış ayarlanması (sensitive data logging).
- Log pipeline'ında redaction veya filtering olmaması.

Savunma & İyileştirme:
- Uygulama kodunda sensitive data logging'i engelleyin.
- Log pipeline'da redaction/PII masking uygulayın.
- Secrets için K8s `Secret` kullanın ve RBAC ile erişimi kısıtlayın.
- Audit logları ve SIEM ile anormallikleri izleyin.

İyileştirme adımları:
- Bulunan secret'ı rotate edin.
- İlgili indexleri/erişimleri kısıtlayın.
- Deploy pipeline'da secret scanning (trivy/matches) ekleyin.
