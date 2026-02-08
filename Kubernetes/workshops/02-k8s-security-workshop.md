# Kubernetes Security Workshop — Temel Uygulamalar

Hedefler:
- Pod Security Admission/PSP benzeri kısıtlamaları uygulamak.
- Basit NetworkPolicy ile pod iletişimini sınırlamak.
- RBAC ile least-privilege ilkesini test etmek.

Gereksinimler:
- Çalışır bir cluster (kind/minikube/remote).
- `kubectl` yetkisi (admin veya uygun izinler).

Adımlar:
1. PSAs uygulama (örnek manifest)
- Cluster'ınız PSA destekliyorsa `restricted` profile'ı aktif edin veya aynı mantığı sağlayan admission webhook ekleyin.

2. NetworkPolicy örneği (default deny + izin verilen trafik)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 5432
```

3. RBAC: servis hesabı ve role oluşturup sınırlı izin verin
```bash
kubectl create namespace demo
kubectl create serviceaccount limited -n demo
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n demo
kubectl create rolebinding rb-pod-reader --role=pod-reader --serviceaccount=demo:limited -n demo
```

4. Test: `kubectl auth can-i` ile izinleri doğrulayın
```bash
kubectl auth can-i get pods --as system:serviceaccount:demo:limited -n demo
```

Kontrol noktası:
- `limited` servis hesabı sadece beklenen kaynakları okuyabiliyor mu?
- NetworkPolicy istenmeyen iletişimi engelliyor mu?

Genişletme (challenge):
- OPA/Gatekeeper ile bir kural yazın: `runAsNonRoot` olmayan podları reddetsin.
- Image admission: sadece belirli registry'lerden gelen image'lara izin verin.
