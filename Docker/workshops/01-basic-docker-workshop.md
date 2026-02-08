# Docker Workshop — Başlangıç

Hedefler:
- Docker kurulumunu doğrulamak.
- Basit bir Dockerfile yazıp image oluşturmak ve çalıştırmak.
- Konteyner yaşam döngüsünü (start/stop/remove) öğrenmek.

Gereksinimler:
- Linux/macOS/Windows üzerinde Docker kurulu.
- Terminal bilgisi (bash/zsh).

Adımlar:
1. Doğrulama
```bash
docker --version
docker run --rm hello-world
```
Beklenen: `hello-world` mesajı görünür.

2. Basit uygulama
- `app/` içinde `app.py` oluştur (örnek: Flask veya basit HTTP server).
- `Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN pip install flask
CMD ["python","app.py"]
```

3. Image oluşturma ve çalıştırma
```bash
docker build -t myapp:local .
docker run -d --name myapp -p 8080:8080 myapp:local
curl http://localhost:8080
```

4. Yönetim
```bash
docker ps
docker logs myapp
docker stop myapp
docker rm myapp
docker rmi myapp:local
```

Kontrol noktası:
- `curl` HTTP 200 döndürüyor mu?
- Container loglarında hata var mı?

Genişletme (challenge):
- Multi-stage build kullanarak final image boyutunu küçültün.
- `docker-compose` ile uygulamayı bir veritabanı ile çalıştırın.

Not: Üretim için container user'ı root olmayan bir kullanıcı ile çalıştırmayı deneyin.