# Images and Containers

## What is a Docker Image?

**Definition:**
A read-only template containing application code, libraries, and dependencies.

**Think of it as:**
- Recipe for a dish
- Blueprint for a house
- Installation package

**Image Characteristics:**
- Immutable (cannot be changed)
- Layered structure
- Shareable and reusable
- Stored in registries

---

## Image Naming Convention

**Format:** `registry/repository:tag`

**Examples:**
```
nginx:latest
  ↓      ↓
name   tag

docker.io/library/nginx:1.21.0
    ↓       ↓      ↓     ↓
registry namespace name version

myregistry.com/team/myapp:v1.2.3
      ↓         ↓     ↓      ↓
   registry  namespace name  tag
```

**Common Tags:**
- `latest` - Most recent version (default)
- `1.21.0` - Specific version
- `stable` - Stable release
- `dev` - Development version

---

## What is a Container?

**Definition:**
A running instance of an image.

**Relationship:**
```
Image (Template)  →  Container (Instance)
Class            →  Object
Recipe           →  Cooked Dish
Blueprint        →  Built House
```

**Container Characteristics:**
- Writable layer on top of image
- Isolated from other containers
- Has its own filesystem, network, processes
- Temporary (data lost when removed)

---

## Image vs Container

**Comparison:**

| Aspect | Image | Container |
|--------|-------|-----------|
| State | Read-only | Writable |
| Purpose | Template | Running instance |
| Lifespan | Permanent | Temporary |
| Count | One | Many (from same image) |
| Storage | MB to GB | Adds few MB |

**Example:**
```bash
# One image
nginx:latest (100 MB)

# Multiple containers from same image
Container 1: nginx-prod    (100 MB + 2 MB)
Container 2: nginx-dev     (100 MB + 1 MB)
Container 3: nginx-staging (100 MB + 3 MB)

# Containers share base image layers
# Only differences stored separately
```

---

## Creating and Running Containers

### Basic Run

```bash
# Run container from image
docker run nginx:latest

# Run in background (-d = detached)
docker run -d nginx:latest

# Run with name
docker run --name my-nginx nginx:latest

# Run and remove when stopped
docker run --rm nginx:latest
```

### Port Mapping

```bash
# Expose container port to host
docker run -p 8080:80 nginx:latest
#          ↓     ↓
#       host  container
# Access: http://localhost:8080
```

### Environment Variables

```bash
# Pass environment variable
docker run -e DB_HOST=localhost postgres

# Multiple variables
docker run \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  postgres
```

### Volume Mounting

```bash
# Mount host directory to container
docker run -v /host/path:/container/path nginx

# Mount current directory
docker run -v $(pwd):/app myapp
```

---

## Container Lifecycle

**States:**
```
Created → Running → Paused → Stopped → Removed
```

**Commands:**
```bash
# Create container (doesn't start)
docker create nginx:latest

# Start created/stopped container
docker start container_id

# Pause running container
docker pause container_id

# Unpause
docker unpause container_id

# Stop running container
docker stop container_id

# Kill container (force stop)
docker kill container_id

# Remove stopped container
docker rm container_id

# Remove running container (force)
docker rm -f container_id
```

---

## Working with Images

### Pull Images

```bash
# Pull from Docker Hub
docker pull nginx:latest

# Pull specific version
docker pull nginx:1.21.0

# Pull from private registry
docker pull myregistry.com/myapp:v1
```

### List Images

```bash
# List all local images
docker images

# Show image details
docker image inspect nginx:latest

# Show image history (layers)
docker history nginx:latest
```

### Remove Images

```bash
# Remove unused images
docker image prune

# Remove specific image
docker rmi nginx:latest

# Force remove (even if containers using it)
docker rmi -f nginx:latest
```

---

## Working with Containers

### Inspect Containers

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Show container details
docker inspect container_id

# Show container logs
docker logs container_id

# Follow logs in real-time
docker logs -f container_id
```

### Execute Commands in Container

```bash
# Run command in running container
docker exec container_id ls /app

# Interactive shell
docker exec -it container_id bash

# As specific user
docker exec -u root container_id whoami
```

### Copy Files

```bash
# Copy from host to container
docker cp /host/file.txt container_id:/container/path/

# Copy from container to host
docker cp container_id:/container/file.txt /host/path/
```

---

## Practical Example

**Scenario:** Run a simple web server

```bash
# 1. Pull nginx image
docker pull nginx:latest

# 2. Run container
docker run -d \
  --name my-web-server \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:latest

# 3. Create index.html
echo "<h1>Hello Docker!</h1>" > html/index.html

# 4. Access in browser
# http://localhost:8080

# 5. View logs
docker logs my-web-server

# 6. Stop and remove
docker stop my-web-server
docker rm my-web-server
```

---

## Image Registries

### Docker Hub (Public)

```bash
# Pull official image
docker pull nginx

# Pull user image
docker pull username/repository:tag

# Push to Docker Hub
docker login
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest
```

### Private Registry

```bash
# Pull from private registry
docker login myregistry.com
docker pull myregistry.com/myapp:v1

# Push to private registry
docker tag myapp:v1 myregistry.com/myapp:v1
docker push myregistry.com/myapp:v1
```

---

## Common Patterns

### Temporary Container

```bash
# Run and auto-remove after exit
docker run --rm -it ubuntu:22.04 bash
```

### Background Service

```bash
# Run database in background
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:14
```

### One-time Task

```bash
# Run script and exit
docker run --rm \
  -v $(pwd):/workspace \
  python:3.9 \
  python /workspace/process.py
```

---

## Key Takeaways

1. **Images are templates** - Read-only, immutable
2. **Containers are instances** - Running, writable
3. **One image → Many containers** - Efficient resource usage
4. **Containers are temporary** - Data lost when removed (use volumes)
5. **Images stored in registries** - Like GitHub for containers

**Next:** Learn how to build your own images with Dockerfile.
