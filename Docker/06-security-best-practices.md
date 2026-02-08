# Security Best Practices

## 1. Use Minimal Base Images

### Problem: Full OS Images

```dockerfile
# 400 MB, thousands of packages
FROM ubuntu:22.04
```

**Issues:**
- Large attack surface
- Unnecessary packages
- More vulnerabilities
- Slower downloads

### Solution: Minimal Images

**Option 1: Alpine Linux**
```dockerfile
# 5 MB, minimal packages
FROM alpine:3.18
```

**Option 2: Distroless**
```dockerfile
# No shell, no package manager
FROM gcr.io/distroless/python3
```

**Comparison:**

| Base Image | Size | Shell | Package Manager | Packages |
|------------|------|-------|-----------------|----------|
| ubuntu:22.04 | 80 MB | ✓ | ✓ | ~100 |
| alpine:3.18 | 5 MB | ✓ | ✓ | ~15 |
| distroless | 20 MB | ✗ | ✗ | ~5 |

**Example:**
```dockerfile
# Before
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
COPY app.py .
CMD ["python3", "app.py"]
# Size: 400 MB, 50 vulnerabilities

# After
FROM python:3.9-alpine
COPY app.py .
CMD ["python", "app.py"]
# Size: 50 MB, 2 vulnerabilities
```

---

## 2. Run as Non-Root User

### The Pattern

```dockerfile
FROM python:3.9-alpine

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /app

# Install as root (if needed)
RUN apk add --no-cache curl

# Copy with correct ownership
COPY --chown=appuser:appuser . .

# Switch to non-root
USER appuser

# Runs as appuser
CMD ["python", "app.py"]
```

### Verification

```bash
# Check user
docker run --rm myapp whoami
# appuser

# Try root command (should fail)
docker run --rm myapp apt-get update
# Error: Permission denied
```

---

## 3. Multi-Stage Builds

### Problem: Build Tools in Production

```dockerfile
# All in one - Bad
FROM node:18
COPY package*.json ./
RUN npm install          # Dev + Prod deps
COPY . .
RUN npm run build        # Build tools included
CMD ["node", "dist/server.js"]
# Size: 500 MB with dev dependencies
```

### Solution: Multi-Stage

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

# Non-root user
RUN adduser -D appuser
USER appuser

CMD ["node", "dist/server.js"]
# Size: 150 MB, no build tools
```

**Benefits:**
- 70% size reduction
- No build tools
- Fewer vulnerabilities
- Faster deployments

---

## 4. Never Store Secrets in Images

### Wrong Approaches

```dockerfile
# WRONG 1: Hardcoded
ENV DB_PASSWORD=secret123

# WRONG 2: Copied file
COPY .env /app/

# WRONG 3: ARG (still visible in history)
ARG API_KEY=secret
ENV API_KEY=$API_KEY
```

### Right Approaches

**Method 1: Runtime Environment Variables**
```bash
docker run -e DB_PASSWORD=secret myapp
```

**Method 2: Docker Secrets**
```bash
echo "secret" | docker secret create db_pass -
docker service create --secret db_pass myapp
```

**Method 3: Build Secrets (BuildKit)**
```dockerfile
# Dockerfile
RUN --mount=type=secret,id=apikey \
    API_KEY=$(cat /run/secrets/apikey) ./configure.sh
```

```bash
# Build
echo "secret" | docker build --secret id=apikey -t myapp .
```

**Method 4: External Secrets Manager**
```python
# app.py
import os
import boto3

def get_secret():
    client = boto3.client('secretsmanager')
    return client.get_secret_value(SecretId='prod/db')['SecretString']
```

---

## 5. Scan Images Regularly

### During Build

```bash
# Scan before push
docker build -t myapp:v1 .
trivy image --severity HIGH,CRITICAL myapp:v1

# If vulnerabilities found, build fails
trivy image --exit-code 1 --severity CRITICAL myapp:v1
```

### In CI/CD

```yaml
# .gitlab-ci.yml
security_scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE
```

### Regular Monitoring

```bash
# Scan production images weekly
trivy image production-registry/myapp:v1.2.3

# Scan all images
docker images --format "{{.Repository}}:{{.Tag}}" | \
  xargs -I {} trivy image {}
```

---

## 6. Use HEALTHCHECK

### Why It Matters

Without healthcheck, Docker only knows if process is running, not if app is healthy.

**Process running but app broken:**
```bash
docker ps
# Container shows as "Up" but app is down
```

### Implementation

```dockerfile
FROM python:3.9-alpine

COPY app.py .

# Add healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8080/health')" \
  || exit 1

CMD ["python", "app.py"]
```

**In app.py:**
```python
@app.route('/health')
def health():
    # Check database, cache, etc.
    if database.is_connected():
        return {"status": "healthy"}, 200
    return {"status": "unhealthy"}, 503
```

---

## 7. Drop Capabilities

### Default Capabilities (Too Many)

```bash
docker run myapp
# Has 14+ capabilities by default
```

### Drop All, Add Only What's Needed

```bash
# Drop all
docker run --cap-drop=ALL myapp

# Add only specific ones
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

**Kubernetes:**
```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

---

## 8. Read-Only Filesystem

### Why

Prevents malware from writing to filesystem.

```bash
# Make filesystem read-only
docker run --read-only myapp
```

**With writable temp directory:**
```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  myapp
```

**Kubernetes:**
```yaml
securityContext:
  readOnlyRootFilesystem: true

volumeMounts:
  - name: tmp
    mountPath: /tmp
```

---

## 9. Set Resource Limits

### Prevent Resource Exhaustion

```bash
docker run \
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="1.0" \
  --pids-limit=100 \
  myapp
```

**Kubernetes:**
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

## 10. Use .dockerignore

### Prevent Accidental Inclusion

**.dockerignore:**
```
# Version control
.git
.gitignore

# Dependencies
node_modules
__pycache__
*.pyc

# Secrets
.env
.env.*
secrets/
*.key
*.pem

# IDE
.vscode
.idea

# Build artifacts
dist/
build/
*.log

# OS
.DS_Store
Thumbs.db
```

---

## Secure Dockerfile Template

**Complete example with all best practices:**

```dockerfile
# Use minimal, specific version
FROM python:3.9-alpine

# Install security updates
RUN apk upgrade --no-cache

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# Set working directory
WORKDIR /app

# Copy and install dependencies first (caching)
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Resource limits metadata
LABEL max-memory="512m" max-cpu="1.0"

# Expose only necessary port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" \
  || exit 1

# Run application
CMD ["python", "app.py"]
```

**Build and run securely:**

```bash
# Build
docker build -t myapp:v1 .

# Scan
trivy image --severity HIGH,CRITICAL myapp:v1

# Run with security options
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --memory="512m" \
  --cpus="1.0" \
  -p 8080:8080 \
  myapp:v1
```

---

## Security Checklist

**Image Building:**
- [ ] Use minimal base image (Alpine/Distroless)
- [ ] Specific version tags (not `latest`)
- [ ] Run as non-root user
- [ ] Multi-stage builds
- [ ] No secrets in image
- [ ] .dockerignore configured
- [ ] HEALTHCHECK defined
- [ ] Scan with Trivy

**Runtime:**
- [ ] Drop all capabilities
- [ ] Read-only filesystem
- [ ] Resource limits set
- [ ] No privileged mode
- [ ] Network policies applied
- [ ] Minimal volume mounts

**Operations:**
- [ ] Regular vulnerability scans
- [ ] Image signing enabled
- [ ] SBOM generated
- [ ] Automated updates
- [ ] Security monitoring

---

## Anti-Patterns to Avoid

**1. Using latest tag**
```dockerfile
FROM python:latest  # BAD
FROM python:3.9-alpine  # GOOD
```

**2. Running as root**
```dockerfile
CMD ["python", "app.py"]  # BAD (runs as root)
USER appuser
CMD ["python", "app.py"]  # GOOD
```

**3. Installing unnecessary packages**
```dockerfile
RUN apt-get install -y build-essential git vim  # BAD
RUN apk add --no-cache curl  # GOOD (only what's needed)
```

**4. Not cleaning up**
```dockerfile
RUN apt-get update
RUN apt-get install -y curl  # BAD (large cache left)

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*  # GOOD
```

**5. Copying everything**
```dockerfile
COPY . .  # BAD (without .dockerignore)
# Create .dockerignore first, then COPY
```

---

## Key Takeaways

1. **Minimal is secure** - Less code, less vulnerabilities
2. **Non-root always** - Limits damage from breaches
3. **Scan everything** - Automate security checks
4. **Multi-stage builds** - Separate build from runtime
5. **No secrets in images** - Use runtime injection
6. **Defense in depth** - Multiple security layers
7. **Automate security** - Integrate into CI/CD

**Next:** Learn runtime security and monitoring.
