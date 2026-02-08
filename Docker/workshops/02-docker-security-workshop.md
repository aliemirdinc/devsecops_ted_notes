# Docker Security Workshop — Temel Güvenlik

Hedefler:
- Image tarama ile zafiyetleri bulmak (Trivy örneği).
- Dockerfile güvenli yazım pratikleri.
- Gizli bilgileri (secrets) imajlara/katmanlara sızdırmaktan kaçınmak.

Gereksinimler:
- `trivy` yüklü (veya CI içinde çalıştırma bilgisi).

Adımlar:
1. Trivy ile tarama
```bash
trivy image --severity HIGH,CRITICAL --no-progress myapp:local
```
Beklenen: HIGH/CRITICAL bulunursa liste gösterilir.

2. Dockerfile hardening
- Tek satırda `RUN` birden fazla komut kullanıp temizleme yapın.
- `--no-install-recommends` ve minimal base image kullanın.
- Multi-stage build ile geliştirme bağımlılıklarını ayırın.

Örnek: multi-stage
```dockerfile
FROM python:3.11-slim AS build
WORKDIR /app
COPY requirements.txt ./
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=build /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . /app
CMD ["python","/app/app.py"]
```

3. Secrets önleme
- `.dockerignore` kullanın.
- `ARG` parametreler ve build-time secrets kullanın (`--secret` mekanizmaları`).
- CI'da secret inject edip image içine yazmayın.

Kontrol noktası:
- Image taraması temiz mi?
- Image boyutu kabul edilebilir mi?

Genişletme (challenge):
- CI pipeline'a Trivy ekleyip başarısızlık eşiğini yapılandırın.
- SBOM oluşturun ve SBOM ile bir tarama kıyaslaması yapın.
