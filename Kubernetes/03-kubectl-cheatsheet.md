# kubectl Cheat-sheet

## Setup
- `kubectl config view` — show kubeconfig
- `kubectl config current-context` — show active cluster/context
- `kubectl config use-context my-cluster` — switch context

## Basic resource inspection
- `kubectl get nodes` — list nodes
- `kubectl get pods -A` — list all pods across namespaces
- `kubectl get svc` — list services in current ns
- `kubectl describe pod my-pod` — detailed pod info
- `kubectl logs pod/my-pod` — pod logs (single container)
- `kubectl logs -c container-name pod/my-pod` — specific container

## Deploying & updating
- `kubectl apply -f manifest.yaml` — create/update resources declaratively
- `kubectl delete -f manifest.yaml` — delete resources
- `kubectl rollout status deployment/my-dep` — check rollout progress
- `kubectl rollout undo deployment/my-dep` — rollback a deployment

## Scaling & troubleshooting
- `kubectl scale deployment/my-dep --replicas=5` — scale
- `kubectl top pod` — resource usage (requires metrics-server)
- `kubectl exec -it pod/my-pod -- /bin/sh` — open shell to pod
- `kubectl port-forward svc/my-svc 8080:80` — forward local port to service

## Labels & selectors
- `kubectl label pod my-pod env=staging` — add label
- `kubectl get pods -l env=staging` — list pods by label

## Namespaces
- `kubectl create namespace ns1`
- `kubectl get pods -n ns1`
- `kubectl config set-context --current --namespace=ns1`

## Shortcuts
- Alias `k` → `kubectl` (add to shell config): `alias k=kubectl`
- Use `-o yaml` or `-o json` for raw output

## Common patterns
- Create resources and watch:
  `kubectl apply -f app.yaml && kubectl get pods -w`
- Debug failing pods:
  `kubectl describe pod pod-name` → check events
  `kubectl logs pod/pod-name --previous` → logs from previous container

---

Next: Security & best practices for Kubernetes.