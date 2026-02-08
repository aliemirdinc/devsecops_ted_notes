# Runtime Security

## What is Runtime Security?

**Definition:**
Security measures applied when containers are actually running in production.

**Why it matters:**
- Build-time security is not enough
- Threats emerge after deployment
- Containers can be exploited at runtime
- Need continuous monitoring

---

## Secure Container Execution

### Basic Security Flags

```bash
docker run \
  --read-only \                    # Read-only filesystem
  --cap-drop=ALL \                 # Drop all capabilities
  --security-opt=no-new-privileges \  # Prevent privilege escalation
  --user 1000:1000 \              # Run as non-root
  --memory="512m" \               # Memory limit
  --cpus="1.0" \                  # CPU limit
  --pids-limit=100 \              # Process limit
  myapp:v1
```

### Explanation of Each Flag

**--read-only**
```bash
# Attacker can't write malware
docker run --read-only nginx

# With writable temp
docker run --read-only --tmpfs /tmp nginx
```

**--cap-drop=ALL**
```bash
# Remove all Linux capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Common capabilities to add:
# NET_BIND_SERVICE - Bind to ports < 1024
# CHOWN - Change file ownership
# DAC_OVERRIDE - Bypass file permissions
```

**--security-opt=no-new-privileges**
```bash
# Prevents gaining more privileges
docker run --security-opt=no-new-privileges myapp
```

**--user**
```bash
# Run as specific user
docker run --user 1000:1000 myapp

# Or use name
docker run --user appuser:appgroup myapp
```

---

## Kubernetes Security Context

### Pod-Level Security

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:v1
    
    # Container-level security
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
    
    # Resource limits
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "256Mi"
        cpu: "250m"
```

### Security Explained

**runAsNonRoot**
```yaml
# Fails if image tries to run as root
runAsNonRoot: true
```

**runAsUser**
```yaml
# Force specific UID
runAsUser: 1000
```

**allowPrivilegeEscalation**
```yaml
# Prevent processes from gaining more privileges
allowPrivilegeEscalation: false
```

**capabilities**
```yaml
# Drop all Linux capabilities
capabilities:
  drop:
    - ALL
```

**seccompProfile**
```yaml
# Restrict system calls
seccompProfile:
  type: RuntimeDefault  # Use default profile
```

---

## Seccomp Profiles

### What is Seccomp?

System calls filter that restricts which kernel functions containers can use.

### Default Profile

```bash
# Use default seccomp profile
docker run --security-opt seccomp=default.json myapp
```

### Custom Profile

**seccomp-profile.json:**
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": [
        "chmod",
        "chown",
        "unshare",
        "mount"
      ],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

```bash
# Apply custom profile
docker run --security-opt seccomp=seccomp-profile.json myapp
```

---

## AppArmor and SELinux

### AppArmor (Ubuntu/Debian)

**Check if loaded:**
```bash
docker run --rm apparmor_status
```

**Use AppArmor profile:**
```bash
docker run --security-opt apparmor=docker-default myapp
```

**Kubernetes:**
```yaml
annotations:
  container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

### SELinux (RHEL/CentOS)

**Use SELinux:**
```bash
docker run --security-opt label=type:container_t myapp
```

**Kubernetes:**
```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

---

## Network Security

### Network Isolation

```bash
# Create isolated network
docker network create --driver bridge isolated-net

# Run containers on isolated network
docker run --network isolated-net app1
docker run --network isolated-net app2

# app1 and app2 can communicate
# But isolated from other containers
```

### Network Policies (Kubernetes)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

**Result:** Only frontend can access backend.

---

## Runtime Monitoring

### Container Logs

```bash
# View logs
docker logs container_id

# Follow logs
docker logs -f container_id

# Last 100 lines
docker logs --tail 100 container_id

# Since timestamp
docker logs --since "2024-01-01T00:00:00" container_id
```

### Process Monitoring

```bash
# View running processes
docker exec container_id ps aux

# Top processes
docker top container_id

# Resource usage
docker stats container_id
```

### Security Events

```bash
# Audit container events
docker events --filter event=create
docker events --filter event=start
docker events --filter event=die

# Filter by container
docker events --filter container=myapp
```

---

## Intrusion Detection

### Falco (Runtime Security)

**What it does:**
- Monitors system calls
- Detects suspicious behavior
- Alerts on security violations

**Example Rules:**
```yaml
# Detect shell in container
- rule: Shell in Container
  condition: spawned_process and container and shell_procs
  output: Shell spawned (user=%user.name container=%container.name)
  priority: WARNING

# Detect sensitive file access
- rule: Read Sensitive Files
  condition: open_read and sensitive_files
  output: Sensitive file opened (file=%fd.name user=%user.name)
  priority: CRITICAL
```

**Deployment:**
```bash
# Install Falco
helm install falco falcosecurity/falco
```

---

## Vulnerability Scanning at Runtime

### Continuous Scanning

```bash
# Scan running containers
trivy image $(docker ps --format "{{.Image}}")

# Automated checks
while true; do
  docker ps --format "{{.Image}}" | \
    xargs -I {} trivy image --severity CRITICAL {}
  sleep 3600  # Every hour
done
```

### Kubernetes Admission Control

**Scan before deployment:**

```yaml
# OPA Gatekeeper policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedImages
metadata:
  name: allowed-images
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    # Only allow scanned images
    allowedImages:
      - "registry.company.com/scanned/*"
```

---

## Incident Response

### When Attack Detected

**1. Isolate Container:**
```bash
# Stop container
docker stop malicious_container

# Or disconnect from network
docker network disconnect bridge malicious_container
```

**2. Collect Evidence:**
```bash
# Save logs
docker logs malicious_container > incident.log

# Export filesystem
docker export malicious_container > forensics.tar

# Inspect configuration
docker inspect malicious_container > config.json
```

**3. Analyze:**
```bash
# Extract filesystem
tar -xf forensics.tar -C /tmp/analysis

# Check for malware
clamscan -r /tmp/analysis

# Review processes
docker top malicious_container
```

**4. Remediate:**
```bash
# Rebuild from clean image
docker build -t myapp:patched .

# Scan before deploy
trivy image myapp:patched

# Rotate secrets
kubectl delete secret app-secrets
kubectl create secret generic app-secrets --from-file=new-creds

# Deploy patched version
kubectl rollout restart deployment/myapp
```

---

## Best Practices

### 1. Least Privilege

```bash
# Minimal permissions
docker run \
  --cap-drop=ALL \
  --read-only \
  --security-opt=no-new-privileges \
  myapp
```

### 2. Network Segmentation

```bash
# Separate networks
docker network create frontend
docker network create backend

docker run --network frontend web
docker run --network backend db
```

### 3. Resource Limits Always

```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 4. Regular Updates

```bash
# Update base images weekly
docker pull python:3.9-alpine
docker build -t myapp:latest .

# Scan for vulnerabilities
trivy image myapp:latest

# Deploy if clean
kubectl set image deployment/myapp app=myapp:latest
```

### 5. Monitor Everything

```bash
# Centralized logging
docker run --log-driver=syslog myapp

# Metrics collection
docker run --label=prometheus.scrape=true myapp
```

---

## Runtime Security Checklist

**Before Running:**
- [ ] Image scanned for vulnerabilities
- [ ] Runs as non-root user
- [ ] Resource limits defined
- [ ] Network policy configured

**During Runtime:**
- [ ] Read-only filesystem enabled
- [ ] Capabilities dropped
- [ ] Seccomp profile applied
- [ ] Logs monitored
- [ ] Alerts configured

**Continuous:**
- [ ] Regular vulnerability scans
- [ ] Security event monitoring
- [ ] Intrusion detection active
- [ ] Incident response plan ready
- [ ] Regular security audits

---

## Secure Run Command Template

```bash
docker run \
  --name myapp \
  --detach \
  --restart unless-stopped \
  \
  # User
  --user 1000:1000 \
  \
  # Security
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --security-opt=seccomp=default.json \
  \
  # Resources
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="1.0" \
  --pids-limit=100 \
  \
  # Network
  --network=isolated \
  --publish 8080:8080 \
  \
  # Monitoring
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  \
  # Health
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  \
  myapp:v1
```

---

## Key Takeaways

1. **Build-time security ≠ Runtime security**
2. **Always use security flags** (read-only, cap-drop, etc.)
3. **Monitor continuously** - Threats emerge after deployment
4. **Least privilege principle** - Minimal permissions
5. **Network isolation** - Segment your services
6. **Resource limits** - Prevent DoS
7. **Incident response plan** - Be ready for attacks

**Next:** Learn CI/CD security integration.
