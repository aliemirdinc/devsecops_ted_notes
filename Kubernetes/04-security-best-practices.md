# Kubernetes Security & Best Practices (Concise)

## Cluster hardening
- Use RBAC with least privilege.
- Enable Pod Security Admission (PSA) or equivalent policies to deny privileged containers.
- Restrict API server access (API server endpoint behind private network or bastion).

## Pod & runtime security
- Don't run containers as root; set `securityContext.runAsNonRoot: true`.
- Use read-only root filesystem when possible (`readOnlyRootFilesystem: true`).
- Limit capabilities and use `capabilities.drop: ["ALL"]` and add back only needed ones.
- Set resource requests and limits to avoid resource contention and DoS.

## Image & supply chain security
- Scan images (Trivy, Clair, Grype) in CI before deploy.
- Pin image digests in manifests (`image: myapp@sha256:...`) to prevent silent updates.
- Use minimal base images and multi-stage builds to avoid unnecessary packages.
- Use SBOMs and provenance (e.g., Sigstore/fulcio/cosign) to verify origin.

## Secrets
- Use Kubernetes Secrets (backed by KMS) or external secret stores (Vault, AWS Secrets Manager).
- Avoid mounting secrets as plaintext files to broad-access volumes.

## Network policy
- Use NetworkPolicies to restrict pod-to-pod communication; default deny where possible.
- Segment workloads by namespace and labels.

## Admission & policy
- Use admission controllers and OPA/Gatekeeper to enforce policies at create/update time.
- Enforce image admission policy (allowlist registries, signed images).

## Monitoring & incident response
- Collect audit logs and forward to centralized SIEM/ELK.
- Monitor for suspicious activity: unexpected image pulls, exec into pods, high error rates.
- Prepare incident runbooks: cordon/drain nodes, isolate namespaces, revoke tokens.

---

Next: Examples for canary with Kubernetes and simple manifests.