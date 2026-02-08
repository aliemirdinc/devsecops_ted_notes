# Kubernetes Workshop — Başlangıç (kind/minikube)

Hedefler:
- Lokal bir cluster oluşturmak (`kind` veya `minikube`).
- Basit bir uygulamayı deploy etmek, service ile eriştirmek, ölçeklemek.
- Rolling update ve rollback pratiği yapmak.

Gereksinimler:
- `kubectl` kurulu.
- `kind` veya `minikube` kurulu.

Adımlar:
1. Cluster oluşturma (kind örneği)
```bash
kind create cluster --name workshop
kubectl cluster-info --context kind-workshop
```

2. Deploy basit app (nginx)
```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --type=NodePort
kubectl get pods,svc
kubectl port-forward svc/web 8080:80
curl http://localhost:8080
```

3. Ölçekleme
```bash
kubectl scale deployment/web --replicas=3
kubectl get pods -l app=web
```

4. Rolling update
```bash
kubectl set image deployment/web nginx=nginx:1.23.0
kubectl rollout status deployment/web
# rollback
kubectl rollout undo deployment/web
```

Kontrol noktası:
- Service'e erişebiliyor musunuz?
- Rollout sorunsuz tamamlandı mı?

Genişletme (challenge):
- Basit readiness/liveness probe ekleyin ve rollout davranışını gözlemleyin.
