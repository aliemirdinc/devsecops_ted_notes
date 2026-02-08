# Next Steps & References

- Learn-by-doing: create a small cluster with `kind` or `minikube` and apply the canary example.
- Useful tools:
  - `kubectl`, `k9s` for inspection
  - `skaffold` / `tilt` for local iterative development
  - `helm` for templating and releases
  - `istio`, `linkerd` for traffic shaping and observability
  - `cert-manager` and `cosign`/`sigstore` for provenance and signing

- Suggested exercises:
  1. Deploy a sample app with `helm` and add resource limits and probes.
  2. Implement network policies to restrict cross-namespace traffic.
  3. Add an admission webhook (OPA/Gatekeeper) to deny privileged containers.

References:
- Kubernetes docs: https://kubernetes.io/docs
- Trivy: https://aquasecurity.github.io/trivy
- Sigstore/cosign: https://sigstore.dev
