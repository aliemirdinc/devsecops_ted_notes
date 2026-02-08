# Kubernetes Architecture — Pod, Deployment, Service, Node

## Pod
- Smallest deployable unit in K8s: one or more containers that share network and storage.
- Co-located containers communicate over localhost and share volumes.
- Pods are ephemeral; controllers recreate pods to maintain desired state.

## Deployment
- Controller that manages ReplicaSets and pods for stateless apps.
- Declarative updates: rolling updates, rollbacks, strategy tuning.
- Example manifest fields: `replicas`, `selector`, `template`, `strategy`.

## Service
- Abstraction that defines a logical set of Pods and a policy to access them.
- Types: `ClusterIP` (internal), `NodePort` (expose on node), `LoadBalancer` (cloud LB), `ExternalName`.
- Service uses selectors to forward traffic to pod endpoints; integrates with Endpoints / EndpointSlices.

## Node
- Worker machine (VM or physical) running kubelet and container runtime (containerd/docker).
- Node resources determine scheduling decisions (CPU/memory/ephemeral-storage).

## Controller Types (quick)
- `ReplicaSet`: ensures a specified number of pod replicas.
- `Deployment`: higher-level API for stateless apps (manages ReplicaSets).
- `StatefulSet`: ordere, stable network IDs, persistent storage for stateful apps.
- `DaemonSet`: run a copy of a pod on every (or selected) node(s).
- `Job` / `CronJob`: batch/one-off jobs; scheduled jobs.

## Networking model
- Every Pod gets its own IP; pods can reach each other without NAT.
- Cluster networking is pluggable via CNI (Calico, Flannel, Weave, Cilium).

---

Next: kubectl cheat-sheet and examples.