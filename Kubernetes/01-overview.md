# Kubernetes & Containers — Overview

## Container = lightweight VM (conceptually)
- Containers package app + runtime + libs into a portable unit.
- Compared to VMs: much smaller, faster to start, share host kernel (not a full guest OS).
- Think: "lightweight process sandbox" rather than full VM.

## Container Management System
- Kubernetes (K8s) is a container orchestration system: deploys, schedules, and manages containers across nodes.
- Responsibilities: placement, scaling, service discovery, health checking, rolling updates, and recovery.

## Granular Management (what K8s gives you)
- Declarative state: describe desired state (manifests) and K8s converges to it.
- Labels & selectors: group and target subsets of pods.
- Resource controls: CPU/memory requests & limits, QoS.
- RBAC and admission controls for policy enforcement.

## Core operational features
- Scaling: increase/reduce replicas to handle load.
- Failover / Self-healing: unhealthy pods are restarted/recreated automatically.
- Load balancing: Service objects route traffic to healthy pod endpoints.
- Auto-restart / auto-replace: controllers maintain desired replica counts.

## Example usage: Canary Deployment (simple, no service mesh)
Goal: send a small percentage of traffic to a new version and collect metrics.

1. Deploy stable `ratings-v1` with N replicas.
2. Deploy `ratings-v2` alongside with small replica count (M).
3. Traffic split ≈ M / (N+M). Example: N=9, M=1 → ~10% to v2.

Commands (concept):
```bash
# deploy v1
kubectl apply -f ratings-v1-deployment.yaml

# deploy v2 (canary)
kubectl apply -f ratings-v2-deployment.yaml

# adjust replicas to tune percentage
kubectl scale deployment/ratings-v1 --replicas=9
kubectl scale deployment/ratings-v2 --replicas=1

# collect metrics on v2 (logs, tracing, metrics)
kubectl logs -l app=ratings,version=v2
# once confident -> increase v2 replicas, then promote and remove v1
kubectl scale deployment/ratings-v2 --replicas=10
kubectl scale deployment/ratings-v1 --replicas=0
```
Notes:
- For precise weighted traffic, use an ingress controller or service mesh (Istio/Linkerd) that supports weighted routing.
- Always monitor latency, error rate, and business metrics during canary.

---

Next: K8s architecture (pods, deployments, services, nodes).