---
name: hermes-dind-patterns
description: Docker-in-Docker workflows for the Hermes multi-tenant platform — sub-container lifecycle, image building, networking, volumes, and service composition.
version: 1.0.0
author: Hermes4Friends
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [docker, dind, containerization, infrastructure]
    related_skills: [hermes-container-environment]
---

# Docker-in-Docker Patterns

You have access to a Docker daemon at `tcp://dind:2375`. You can do anything Docker can do — build images, run containers, manage networks and volumes. The dind container runs privileged, so you effectively have full Docker power inside an isolated executor.

## Prerequisites

```bash
export DOCKER_HOST=tcp://dind:2375
```

## Fundamental: You Are Root Inside Containers

The Hermes agent container runs as UID 10000 non-root. But **every container you launch via dind starts as root inside that container**. This is the escape hatch for anything requiring privileges:

```bash
# Need to install system packages? Launch a container.
# Need to bind to port 80? Launch a container.
# Need to write to /etc/? Launch a container.
docker run --rm -it alpine sh  # You are now root.
```

## Sub-Container Lifecycle

### One-Shot: Run and Discard

```bash
docker run --rm alpine echo "done"          # Exits immediately, auto-removed
docker run --rm -v $(pwd):/work alpine sh -c "apk add curl && curl example.com"
```

### Long-Running Service

```bash
docker run -d --name my-service --restart unless-stopped \
  -p 8080:80 \
  -v /home/hermes/data:/data \
  my-image:latest

# Check logs:
docker logs my-service

# Stop/remove:
docker stop my-service && docker rm my-service
```

### Interactive Session

```bash
docker run --rm -it alpine sh               # Interactive shell
docker run --rm -it -v $(pwd):/work alpine sh  # With mounted workspace
```

## Networking: Connect to Your Containers

All containers share `hermes-net`. You can reach them by name:

```bash
docker run -d --name redis redis:alpine
redis-cli -h redis ping           # Works from any container on hermes-net
```

From your Hermes agent container:
```bash
curl http://redis:6379             # Doesn't work — Hermes is on hermes-net, but the service needs a port mapping or you query from inside a container
```

### Pattern: Query Service from Inside a Container

```bash
docker run --rm alpine sh -c "apk add curl && curl http://redis:6379/"
```

### Pattern: Expose to Host Ports

```bash
docker run -d -p 3000:3000 my-app  # Now accessible at localhost:3000
```

The host (Hetzner VM) port mapping lets Cloudflare tunnels and external users reach your service.

## Building Images

### From a Dockerfile

```bash
docker build -t my-app:v1 /home/hermes/project/
```

### From a Python Project

```bash
cat > /home/hermes/project/Dockerfile << 'EOF'
FROM python:3.12-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
EOF

docker build -t my-python-app /home/hermes/project/
docker run -d --name my-python-app -p 8080:8080 my-python-app
```

### Multi-Stage Builds (Keep Images Small)

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Run stage (tiny, no Go toolchain)
FROM alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /server
CMD ["/server"]
```

## Volumes: Persist Data Past Container Life

```bash
# Named volume (managed by Docker):
docker volume create my-data
docker run -d -v my-data:/data --name db postgres:alpine

# Bind mount (direct path on dind filesystem — NOT your Hermes home):
docker run -d -v /var/lib/myapp:/data my-app

# Bind mount your Hermes home (persists past Hermes container restart):
docker run -d -v /home/hermes/data:/data my-app
```

### Important Volume Distinction

- Bind mounts to `/home/hermes/...` → survive Hermes container restart
- Bind mounts to other paths → survive dind restart, lost if dind is rebuilt
- Named volumes → survive dind restart, persist on dind's filesystem

## Environment Variables in Containers

```bash
# Pass single var:
docker run -e MY_KEY=value alpine env

# Pass from your environment:
docker run -e DEEPSEEK_API_KEY alpine env

# Pass a file as env:
docker run --env-file /home/hermes/.env alpine env
```

## Service Composition (Docker Compose Alternative)

Since `docker compose` may not be installed, use shell scripts for multi-container setups:

```bash
#!/bin/sh
# Start a full stack: app + DB + cache

docker run -d --name redis redis:alpine
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:alpine

# Wait for DB:
docker run --rm postgres:alpine sh -c \
  "until pg_isready -h postgres; do sleep 1; done"

docker run -d --name app \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://postgres:secret@postgres:5432/postgres \
  -e REDIS_URL=redis://redis:6379 \
  -v /home/hermes/app:/app \
  my-app
```

## Cleanup

```bash
# Remove all stopped containers:
docker container prune -f

# Remove unused images:
docker image prune -f

# Remove unused volumes (careful!):
docker volume prune -f

# Full cleanup (images + containers + volumes + networks):
docker system prune -af --volumes
```

## Capacity Notes

- The dind container shares the host's resources
- Don't run dozens of heavy containers — you're sharing with other friend agents
- Prefer Alpine-based images (10x smaller than Ubuntu/Debian)
- Use `docker stats` to monitor resource usage
- Containers with `--restart unless-stopped` survive dind restart but NOT host restart
