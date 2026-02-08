# Canary Deployment Example (Manifests)

This is a minimal, pragmatic example showing a manual canary using two Deployments and a Service.

## ratings Service (service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ratings
spec:
  selector:
    app: ratings
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

## ratings-v1 Deployment (ratings-v1-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratings-v1
  labels:
    app: ratings
    version: v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: ratings
      version: v1
  template:
    metadata:
      labels:
        app: ratings
        version: v1
    spec:
      containers:
      - name: ratings
        image: ghcr.io/example/ratings:v1
        ports:
        - containerPort: 8080
```

## ratings-v2 Deployment (ratings-v2-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratings-v2
  labels:
    app: ratings
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratings
      version: v2
  template:
    metadata:
      labels:
        app: ratings
        version: v2
    spec:
      containers:
      - name: ratings
        image: ghcr.io/example/ratings:v2
        ports:
        - containerPort: 8080
```

## Deploy & test
```bash
kubectl apply -f service.yaml
kubectl apply -f ratings-v1-deployment.yaml
kubectl apply -f ratings-v2-deployment.yaml

# watch pods
kubectl get pods -l app=ratings -w

# logs for v2
kubectl logs -l app=ratings,version=v2
```

Notes:
- Traffic split is proportional to pod counts behind the Service.
- For precise routing, use Ingress with canary support or a service mesh.
