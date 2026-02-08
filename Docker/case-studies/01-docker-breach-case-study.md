# Case Study: Container İhlali — Basit Senaryo

Senaryo:
- Geliştirici yanlışlıkla uygulama yapılandırmasında API anahtarını `config/` içinde tuttu ve bu klasör image içine kopyalandı.
- Image public registry'ye push edildi.
- Kötü niyetli bir aktör image'i indirip anahtarı ele geçirdi.

Saldırı adımları (özet):
1. Registry'den image indirildi.
2. `docker run --rm myapp:public cat /app/config/secret.yaml` ile anahtar okundu.
3. Anahtar ile üçüncü taraf servise yetkisiz erişim sağlandı.

Neden oldu:
- Secrets dosyalarının source tree içinde tutulması.
- Image katmanlarına sızdırma (secret'ler image history içinde kaldı).

Savunma & İyileştirme:
- Secrets'ları environment secrets manager (Vault, AWS Secrets Manager) kullanarak saklayın.
- `.gitignore` ve `.dockerignore` ile hassas dosyaları dışlayın.
- Image taraması ve SBOM check ekleyin.
- Registry'ye push etmeden önce `trivy` ile tarama yapın ve otomatik reddetme (CI gate) kurun.
- Mevcut public secret'leri rotasyon ve revocation ile iptal edin.

Öğrenilen dersler:
- Secrets as code anti-pattern'idir.
- History temizliği (eski image katmanları/commits) gerekli olabilir.
- Otomatik tarama ve policy enforcement önemlidir.