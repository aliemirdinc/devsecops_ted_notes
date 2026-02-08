# Dockerfile Basics

## What is a Dockerfile?

**Definition:**
A text file containing instructions to build a Docker image.

**Think of it as:**
- Recipe to cook a dish
- Blueprint to build a house
- Installation script

**File Name:** `Dockerfile` (no extension, capital D)

---

## Basic Dockerfile Structure

```dockerfile
# Start from base image
FROM ubuntu:22.04

# Set working directory
WORKDIR /app

# Copy files
COPY app.py .

# Install dependencies
RUN apt-get update && apt-get install -y python3

# Expose port
EXPOSE 8080

# Run application
CMD ["python3", "app.py"]
```

---

## Core Instructions

### FROM - Base Image

**Purpose:** Specify the starting point for your image.

**Syntax:**
```dockerfile
FROM image:tag
```

**Examples:**
```dockerfile
# Official base images
FROM ubuntu:22.04
FROM python:3.9
FROM node:18-alpine
FROM nginx:latest

# Minimal images
FROM alpine:3.18
FROM scratch  # Empty image (advanced)
```

**Rule:** Must be first instruction in Dockerfile.

---

### RUN - Execute Commands

**Purpose:** Execute commands during image build.

**Syntax:**
```dockerfile
RUN command
RUN ["executable", "param1", "param2"]
```

**Examples:**
```dockerfile
# Install package
RUN apt-get update && apt-get install -y curl

# Create directory
RUN mkdir -p /app/data

# Multiple commands
RUN apt-get update && \
    apt-get install -y python3 && \
    apt-get clean
```

**Best Practice:** Chain commands to reduce layers.

```dockerfile
# BAD - Creates 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# GOOD - Creates 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean
```

---

### COPY - Copy Files

**Purpose:** Copy files from host to image.

**Syntax:**
```dockerfile
COPY source destination
COPY ["source", "destination"]
```

**Examples:**
```dockerfile
# Copy single file
COPY app.py /app/

# Copy directory
COPY ./src /app/src

# Copy multiple files
COPY package.json package-lock.json /app/

# Copy with wildcard
COPY *.py /app/
```

**Tip:** Destination should be absolute path or relative to WORKDIR.

---

### ADD - Advanced Copy

**Purpose:** Like COPY but with extra features.

**Features:**
- Can extract tar files
- Can download from URLs

**Examples:**
```dockerfile
# Extract tar file
ADD archive.tar.gz /app/

# Download from URL (not recommended)
ADD https://example.com/file.txt /app/
```

**Best Practice:** Use COPY instead of ADD unless you need extraction.

---

### WORKDIR - Set Working Directory

**Purpose:** Set the working directory for subsequent instructions.

**Syntax:**
```dockerfile
WORKDIR /path/to/directory
```

**Examples:**
```dockerfile
WORKDIR /app

# Equivalent to cd /app
COPY app.py .        # Copies to /app/app.py
RUN ls               # Lists /app directory

WORKDIR /data
RUN pwd              # Prints /data
```

**Best Practice:** Always use WORKDIR instead of `RUN cd`.

---

### CMD - Default Command

**Purpose:** Specify default command when container starts.

**Syntax:**
```dockerfile
CMD ["executable", "param1", "param2"]  # Preferred
CMD command param1 param2               # Shell form
```

**Examples:**
```dockerfile
# Run Python app
CMD ["python3", "app.py"]

# Run Node app
CMD ["node", "server.js"]

# Run shell script
CMD ["./start.sh"]

# Shell form
CMD python3 app.py
```

**Important:** Only last CMD in Dockerfile takes effect.

---

### ENTRYPOINT - Fixed Command

**Purpose:** Set fixed command that always runs.

**Difference from CMD:**
- ENTRYPOINT: Fixed command
- CMD: Default arguments (can be overridden)

**Examples:**
```dockerfile
# ENTRYPOINT only
ENTRYPOINT ["python3", "app.py"]
# docker run myapp          → python3 app.py

# ENTRYPOINT + CMD
ENTRYPOINT ["python3"]
CMD ["app.py"]
# docker run myapp          → python3 app.py
# docker run myapp test.py  → python3 test.py
```

---

### EXPOSE - Document Ports

**Purpose:** Document which ports the container listens on.

**Syntax:**
```dockerfile
EXPOSE port
EXPOSE port/protocol
```

**Examples:**
```dockerfile
# HTTP server
EXPOSE 8080

# Multiple ports
EXPOSE 8080 8443

# Specify protocol
EXPOSE 53/udp
EXPOSE 80/tcp
```

**Note:** EXPOSE doesn't publish ports. Use `docker run -p` for that.

---

### ENV - Environment Variables

**Purpose:** Set environment variables.

**Syntax:**
```dockerfile
ENV key=value
ENV key1=value1 key2=value2
```

**Examples:**
```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV APP_HOME=/app \
    APP_PORT=8080 \
    LOG_LEVEL=info
```

**Usage in Dockerfile:**
```dockerfile
ENV APP_DIR=/app
WORKDIR $APP_DIR
COPY . $APP_DIR
```

---

### ARG - Build Arguments

**Purpose:** Pass variables at build time.

**Syntax:**
```dockerfile
ARG name
ARG name=default_value
```

**Examples:**
```dockerfile
# Define build argument
ARG VERSION=1.0

# Use in Dockerfile
FROM python:${VERSION}

# Build with custom value
# docker build --build-arg VERSION=3.9 .
```

**Difference ARG vs ENV:**
- ARG: Build-time only
- ENV: Build-time and runtime

---

### USER - Set User

**Purpose:** Specify which user runs the container.

**Syntax:**
```dockerfile
USER username
USER uid:gid
```

**Examples:**
```dockerfile
# Create and switch to non-root user
RUN useradd -m appuser
USER appuser

# Use numeric ID
USER 1000:1000
```

**Security:** Always run as non-root user in production.

---

## Complete Example: Python Application

**Dockerfile:**
```dockerfile
# Use official Python image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements first (for caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m appuser && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

# Run application
CMD ["python", "app.py"]
```

**Build and Run:**
```bash
# Build image
docker build -t myapp:v1 .

# Run container
docker run -d -p 8080:8080 myapp:v1
```

---

## Complete Example: Node.js Application

**Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy app code
COPY . .

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser && \
    chown -R appuser:appuser /app

USER appuser

EXPOSE 3000

CMD ["node", "server.js"]
```

---

## Build Process

**Command:**
```bash
docker build [OPTIONS] PATH
```

**Common Options:**
```bash
# Build with tag
docker build -t myapp:v1 .

# Build with custom Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg VERSION=3.9 -t myapp .

# Build without cache
docker build --no-cache -t myapp .

# Build and show output
docker build --progress=plain -t myapp .
```

---

## .dockerignore File

**Purpose:** Exclude files from build context.

**File:** `.dockerignore`

**Example:**
```
# Version control
.git
.gitignore

# Dependencies
node_modules
__pycache__
*.pyc

# IDE
.vscode
.idea

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Secrets
.env
secrets/
*.key
```

**Benefits:**
- Faster builds
- Smaller images
- Prevents secret leakage

---

## Layer Caching

**How it works:**
Docker caches each layer. If instruction hasn't changed, uses cached layer.

**Optimization:**
```dockerfile
# BAD - Copies everything first
FROM python:3.9
COPY . /app
RUN pip install -r requirements.txt

# If any file changes, pip install runs again

# GOOD - Copy requirements first
FROM python:3.9
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app

# pip install only runs if requirements.txt changes
```

**Order Matters:**
- Put frequently changing instructions last
- Put rarely changing instructions first

---

## Best Practices

1. **Use specific base image tags**
   ```dockerfile
   # BAD
   FROM python:latest
   
   # GOOD
   FROM python:3.9-slim
   ```

2. **Minimize layers**
   ```dockerfile
   # Chain commands
   RUN apt-get update && \
       apt-get install -y curl && \
       apt-get clean
   ```

3. **Copy dependencies first**
   ```dockerfile
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   ```

4. **Run as non-root**
   ```dockerfile
   USER appuser
   ```

5. **Use .dockerignore**
   ```
   node_modules
   .git
   ```

---

## Common Mistakes

**1. Running as root**
```dockerfile
# BAD
CMD ["python", "app.py"]

# GOOD
USER appuser
CMD ["python", "app.py"]
```

**2. Not cleaning up**
```dockerfile
# BAD
RUN apt-get update
RUN apt-get install -y curl

# GOOD
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

**3. Copying everything**
```dockerfile
# BAD
COPY . .

# GOOD - Use .dockerignore
# Then
COPY . .
```

---

## Key Takeaways

1. **Dockerfile = Recipe** for building images
2. **FROM** must be first instruction
3. **RUN** executes during build
4. **CMD** executes when container starts
5. **COPY** files before using them
6. **WORKDIR** sets working directory
7. **USER** runs as non-root (security)
8. **Layer caching** speeds up builds

**Next:** Learn about Docker layers and optimization.
