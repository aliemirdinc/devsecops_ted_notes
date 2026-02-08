# What is Docker?

## The Problem Docker Solves

**Classic Development Issue:**
```
Developer: "It works on my machine!"
Operations: "Well, it doesn't work in production!"
```

**Why this happens:**
- Different OS versions
- Different library versions
- Different configurations
- Missing dependencies

---

## What is a Container?

**Simple Definition:**
A container is a package that contains:
- Your application code
- All dependencies
- Runtime environment
- Configuration files

**Result:** Same container runs identically everywhere (laptop, server, cloud).

---

## Docker vs Virtual Machines

**Virtual Machine Architecture:**
```
┌─────────────────────┐
│   Application       │
│   Libraries         │
│   Guest OS          │  ← Full OS per app
│   Hypervisor        │
│   Host OS           │
│   Hardware          │
└─────────────────────┘
Size: ~GB per app
Boot: Minutes
```

**Container Architecture:**
```
┌─────────────────────┐
│   Application       │
│   Libraries         │
│   Docker Engine     │
│   Host OS           │  ← Shared kernel
│   Hardware          │
└─────────────────────┘
Size: ~MB per app
Boot: Seconds
```

**Key Differences:**

| Feature | Virtual Machine | Container |
|---------|----------------|-----------|
| Size | Gigabytes | Megabytes |
| Startup | Minutes | Seconds |
| Isolation | Full OS | Process level |
| Performance | Slower | Near native |

---

## Core Concepts

### Image
A blueprint/template for containers. Like a class in programming.

**Example:**
```
nginx:latest = Image of nginx web server
```

### Container
A running instance of an image. Like an object instantiated from a class.

**Example:**
```bash
docker run nginx:latest
# Creates and runs a container from nginx image
```

### Dockerfile
A recipe/script to build an image.

**Example:**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3
COPY app.py /app/
CMD ["python3", "/app/app.py"]
```

### Registry
A storage location for images (like GitHub for code).

**Examples:**
- Docker Hub (public)
- Private registries
- Cloud registries (AWS ECR, Google GCR)

---

## How Docker Works

**Build → Ship → Run**

**1. Build:**
```bash
docker build -t myapp:v1 .
# Creates image from Dockerfile
```

**2. Ship:**
```bash
docker push myregistry/myapp:v1
# Uploads image to registry
```

**3. Run:**
```bash
docker run myregistry/myapp:v1
# Downloads and runs container
```

---

## Real-World Analogy

**Shipping Containers vs Docker Containers:**

**Shipping Container:**
- Standard size and format
- Contains goods
- Can be moved by ship, truck, train
- Receiver doesn't care what's inside

**Docker Container:**
- Standard format
- Contains application
- Can run on any system with Docker
- Infrastructure doesn't care what's inside

---

## Basic Commands

**Images:**
```bash
# List images
docker images

# Pull image from registry
docker pull nginx:latest

# Remove image
docker rmi nginx:latest
```

**Containers:**
```bash
# List running containers
docker ps

# List all containers (running + stopped)
docker ps -a

# Run container
docker run nginx:latest

# Stop container
docker stop container_id

# Remove container
docker rm container_id
```

**Build:**
```bash
# Build image from Dockerfile
docker build -t myapp:v1 .

# Build with custom Dockerfile name
docker build -f Dockerfile.prod -t myapp:prod .
```

---

## Why Docker Matters

**For Developers:**
- Consistent development environment
- Easy dependency management
- Fast setup for new team members

**For Operations:**
- Standardized deployment
- Easy scaling
- Predictable behavior

**For Organizations:**
- Faster time to market
- Resource efficiency
- CI/CD automation

---

## Key Takeaway

Docker packages applications with everything they need to run. Same package works everywhere. This solves the "works on my machine" problem forever.

**Next:** Learn about images and containers in detail.
