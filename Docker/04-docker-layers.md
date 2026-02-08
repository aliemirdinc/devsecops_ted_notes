# Docker Layers

## What are Layers?

**Definition:**
Each instruction in Dockerfile creates a new read-only layer in the image.

**Structure:**
```
Final Image
├── Layer 5: CMD ["python", "app.py"]   (metadata only)
├── Layer 4: COPY app.py /app/          (10 KB)
├── Layer 3: RUN pip install flask      (50 MB)
├── Layer 2: COPY requirements.txt      (1 KB)
└── Layer 1: FROM python:3.9            (100 MB)
```

**Total Size:** 100 + 1 + 50 + 10 = ~161 MB

---

## How Layers Work

### Layer Creation

**Each RUN, COPY, ADD creates a new layer:**

```dockerfile
FROM ubuntu:22.04           # Layer 1
RUN apt-get update          # Layer 2
RUN apt-get install curl    # Layer 3
COPY app.py /app/           # Layer 4
CMD ["python", "app.py"]    # Layer 5 (metadata)
```

### Layer Stacking

```
┌─────────────────────┐
│ CMD (Metadata)      │
├─────────────────────┤
│ COPY app.py         │  ← Writable when container runs
├─────────────────────┤
│ RUN install curl    │
├─────────────────────┤  ← All read-only
│ RUN apt-get update  │
├─────────────────────┤
│ FROM ubuntu         │
└─────────────────────┘
```

---

## Layer Sharing

**Efficient Storage:**

```bash
# Image 1
FROM ubuntu:22.04  # 80 MB
RUN apt-get update # 20 MB
Total: 100 MB

# Image 2
FROM ubuntu:22.04  # Shared! 0 MB
RUN apt-get update # Shared! 0 MB
RUN apt-get install nginx # 30 MB
Total: 30 MB (not 130 MB!)

# Total on disk: 130 MB (not 230 MB!)
```

**Benefit:** Multiple images share common base layers.

---

## The Critical Problem: Deletion Doesn't Reduce Size

### Example of the Problem

```dockerfile
FROM ubuntu:22.04

# Layer 1: Add 100 MB file
RUN dd if=/dev/zero of=/bigfile bs=1M count=100

# Layer 2: Delete the file
RUN rm /bigfile
```

**Result:**
- Layer 1 contains bigfile (100 MB)
- Layer 2 marks bigfile as deleted
- **Total image size: Still 100+ MB!**

**Why?** Layers are immutable. Deletion creates a new layer that hides the file, but the original layer still exists.

---

## Secret Leakage via Layers

### Dangerous Pattern

```dockerfile
FROM ubuntu:22.04

# Layer 1: Add credentials
COPY credentials.txt /tmp/

# Layer 2: Use credentials
RUN cat /tmp/credentials.txt > /app/config

# Layer 3: Delete credentials
RUN rm /tmp/credentials.txt
```

**Attack:**
```bash
# Extract image
docker save myapp:latest -o myapp.tar
tar -xf myapp.tar

# Find credentials in Layer 1
find . -name "credentials.txt"
./layer1/tmp/credentials.txt  # Still there!

cat ./layer1/tmp/credentials.txt
# Secrets exposed!
```

---

## Layer Optimization

### Bad: Multiple Layers

```dockerfile
# Creates 4 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean
```

**Result:** 4 layers, larger image

### Good: Single Layer

```dockerfile
# Creates 1 layer
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Result:** 1 layer, smaller image (cleanup actually works!)

---

## Layer Caching Strategy

### How Caching Works

**Build 1:**
```dockerfile
FROM python:3.9              # Cache miss, pull
COPY requirements.txt .      # Cache miss, copy
RUN pip install -r req.txt   # Cache miss, install
COPY app.py .                # Cache miss, copy
```

**Build 2 (after changing app.py):**
```dockerfile
FROM python:3.9              # Cache HIT ✓
COPY requirements.txt .      # Cache HIT ✓
RUN pip install -r req.txt   # Cache HIT ✓
COPY app.py .                # Cache MISS (file changed)
```

**Result:** Only last instruction re-runs. Much faster!

---

### Optimize for Cache

**Bad Order:**
```dockerfile
FROM python:3.9
COPY . .                    # All files copied
RUN pip install -r req.txt  # Re-runs if ANY file changes

# Change app.py → pip install runs again!
```

**Good Order:**
```dockerfile
FROM python:3.9
COPY requirements.txt .     # Only copy what's needed
RUN pip install -r req.txt  # Runs only if req.txt changes
COPY . .                    # Copy rest of files

# Change app.py → pip install uses cache!
```

---

## Inspecting Layers

### View Layer History

```bash
# Show layers
docker history nginx:latest

IMAGE          CREATED        SIZE
abc123         2 days ago     10MB   CMD ["nginx"]
def456         2 days ago     50MB   COPY . /app
ghi789         2 days ago     100MB  RUN apt-get install
```

### Detailed Layer Inspection

```bash
# Save image as tar
docker save nginx:latest -o nginx.tar

# Extract
tar -xf nginx.tar

# View structure
ls -la
# config.json
# manifest.json
# layer1/
# layer2/
# layer3/
```

---

## Multi-Stage Builds

**Problem:** Build tools increase image size.

**Single-Stage (Bad):**
```dockerfile
FROM node:18

# Install dependencies
COPY package*.json ./
RUN npm install  # Dev + Prod dependencies

# Build
COPY . .
RUN npm run build

# Dev dependencies still in final image!
CMD ["node", "dist/server.js"]
```
**Image size: 500 MB**

**Multi-Stage (Good):**
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
COPY package*.json ./
RUN npm install --production  # Only prod deps
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/server.js"]
```
**Image size: 150 MB** (70% reduction!)

---

## Layer Best Practices

### 1. Chain Related Commands

```dockerfile
# Good
RUN apt-get update && \
    apt-get install -y package && \
    apt-get clean
```

### 2. Clean Up in Same Layer

```dockerfile
# Bad - cleanup doesn't reduce size
RUN apt-get install bigpackage
RUN apt-get clean

# Good - cleanup reduces size
RUN apt-get install bigpackage && \
    apt-get clean
```

### 3. Order by Change Frequency

```dockerfile
# Rarely changes
FROM python:3.9
RUN apt-get update

# Changes sometimes
COPY requirements.txt .
RUN pip install -r requirements.txt

# Changes frequently
COPY . .
```

### 4. Use .dockerignore

```
# .dockerignore
node_modules
.git
*.log
.env
```

### 5. Minimize Layer Count

```dockerfile
# Bad: 6 layers
COPY file1 .
COPY file2 .
COPY file3 .
COPY file4 .
COPY file5 .
COPY file6 .

# Good: 1 layer
COPY file1 file2 file3 file4 file5 file6 ./
```

---

## Practical Example

### Before Optimization

```dockerfile
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y curl
RUN apt-get install -y wget

COPY app.py .
COPY config.py .
COPY utils.py .

RUN pip3 install flask
RUN pip3 install requests

RUN apt-get clean
```

**Layers:** 11 layers
**Size:** 400 MB

### After Optimization

```dockerfile
FROM python:3.9-alpine

# Single layer for system packages + cleanup
RUN apk add --no-cache curl wget

# Copy requirements before code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY *.py ./

CMD ["python", "app.py"]
```

**Layers:** 5 layers
**Size:** 100 MB (75% reduction!)

---

## Security Implications

### Secret in Layer Attack

**Vulnerable:**
```dockerfile
RUN echo "API_KEY=secret123" > .env
RUN source .env && ./deploy.sh
RUN rm .env
```

**Attacker extracts .env from layer 1!**

**Secure:**
```dockerfile
# Use build secrets (Docker BuildKit)
RUN --mount=type=secret,id=apikey \
    API_KEY=$(cat /run/secrets/apikey) ./deploy.sh
```

```bash
# Build with secret
echo "secret123" | docker build --secret id=apikey -
```

---

## Key Takeaways

1. **Each instruction creates a layer** (except CMD, ENV metadata)
2. **Layers are immutable** - Deletion doesn't shrink size
3. **Layers are shared** - Save disk space
4. **Order matters** - Put frequently changing instructions last
5. **Chain commands** - Reduce layer count
6. **Clean in same layer** - Cleanup actually works
7. **Never store secrets** - They remain in layers forever
8. **Use multi-stage builds** - Dramatically reduce image size

**Next:** Learn about Docker security risks.
