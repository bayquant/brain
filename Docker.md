---
tags: [docker, containers, devops, cli]
---
# Docker

Docker packages applications into portable, isolated containers. An **image** is the blueprint (read-only); a **container** is a running instance of an image.

---

## CORE CONCEPTS

```
Image (blueprint)
└── Container (running instance)
    ├── Filesystem (copy-on-write layer over the image)
    ├── Network interface
    └── Isolated process namespace

Registry (Docker Hub, ACR, ECR, GCR)
└── Repository (e.g. nginx)
    └── Tag (e.g. nginx:1.27-alpine)
```

- **Volume** — persisted storage that lives outside the container filesystem
- **Network** — virtual bridge between containers; `bridge` (default), `host`, `none`, `overlay`
- **Layer cache** — each `RUN`/`COPY`/`ADD` instruction creates a layer; unchanged layers are reused on rebuild

---

## CLI SETUP

```bash
# Verify installation
docker --version
docker info

# Login to Docker Hub
docker login

# Login to a private registry
docker login <registry-host>
```

---

## IMAGES

```bash
# Pull an image
docker pull nginx:1.27-alpine

# List local images
docker images

# Build an image from a Dockerfile in current directory
docker build -t <name>:<tag> .

# Build with a specific Dockerfile
docker build -f Dockerfile.prod -t <name>:<tag> .

# Tag an existing image
docker tag <source>:<tag> <target>:<tag>

# Push to a registry
docker push <registry>/<name>:<tag>

# Remove an image
docker rmi <image>

# Remove all unused images
docker image prune -a
```

---

## CONTAINERS

```bash
# Run a container (pulls image if missing)
docker run nginx

# Run detached (background)
docker run -d nginx

# Run with a name
docker run -d --name my-nginx nginx

# Run with port mapping  host:container
docker run -d -p 8080:80 nginx

# Run with environment variables
docker run -d -e ENV=production -e DB_URL=postgres://... nginx

# Run with a volume mount
docker run -d -v /host/path:/container/path nginx

# Run with a named volume
docker run -d -v my-data:/var/lib/data nginx

# Run interactively (e.g. for debugging)
docker run -it ubuntu bash

# Run and remove on exit
docker run --rm alpine echo "hello"

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop / start / restart
docker stop <container>
docker start <container>
docker restart <container>

# Remove a stopped container
docker rm <container>

# Remove all stopped containers
docker container prune
```

---

## INSPECT AND DEBUG

```bash
# View logs
docker logs <container>

# Follow logs
docker logs -f <container>

# Tail last 100 lines
docker logs --tail 100 <container>

# Execute a command in a running container
docker exec -it <container> bash

# Inspect container details (JSON)
docker inspect <container>

# Show resource usage
docker stats

# Show running processes inside container
docker top <container>

# Copy files between host and container
docker cp <container>:/path/file.txt ./local/
docker cp ./local/file.txt <container>:/path/
```

---

## VOLUMES

```bash
# Create a named volume
docker volume create my-data

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect my-data

# Remove a volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune
```

---

## NETWORKS

```bash
# List networks
docker network ls

# Create a custom bridge network
docker network create my-network

# Run container on a custom network
docker run -d --network my-network --name app nginx

# Connect a running container to a network
docker network connect my-network <container>

# Inspect a network
docker network inspect my-network
```

Containers on the same custom network can reach each other by **container name** as the hostname.

---

## DOCKERFILE

```dockerfile
# Base image
FROM python:3.12-slim

# Set working directory
WORKDIR /app

# Copy dependency files first (layer cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Non-root user (security best practice)
RUN adduser --disabled-password appuser
USER appuser

# Expose port (documentation only — does not publish)
EXPOSE 8080

# Environment variable defaults
ENV PORT=8080

# Entrypoint vs CMD:
#   ENTRYPOINT — fixed executable, cannot be overridden without --entrypoint
#   CMD        — default args, overridden by anything passed to docker run
ENTRYPOINT ["python"]
CMD ["app.py"]
```

### MULTI-STAGE BUILD

Keeps the final image small by discarding build tools.

```dockerfile
# Stage 1 — build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2 — runtime (only copies built output)
FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### .DOCKERIGNORE

```
.git
.env
node_modules
__pycache__
*.pyc
.venv
dist
```

---

## DOCKER COMPOSE

Defines and runs multi-container applications. All services share a default network and can reach each other by **service name**.

### COMPOSE FILE

```yaml
# compose.yaml  (preferred name; docker-compose.yml also works)
services:

  app:
    build: .                          # build from local Dockerfile
    image: myapp:latest               # tag the built image
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/mydb
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy    # wait for health check to pass
    volumes:
      - ./src:/app/src                # bind mount for dev hot-reload
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pg-data:
```

---

## COMPOSE COMMANDS

```bash
# Start all services (build if needed)
docker compose up

# Start detached
docker compose up -d

# Rebuild images before starting
docker compose up -d --build

# Stop services (keep containers)
docker compose stop

# Stop and remove containers, networks
docker compose down

# Stop and remove containers, networks, AND volumes
docker compose down -v

# View logs for all services
docker compose logs

# Follow logs for a specific service
docker compose logs -f app

# Run a one-off command in a service container
docker compose exec app bash
docker compose run --rm app python manage.py migrate

# Scale a service
docker compose up -d --scale worker=3

# List running services
docker compose ps

# Rebuild a single service
docker compose build app

# Pull latest images
docker compose pull
```

---

## COMPOSE PROFILES

Group optional services behind a profile — only started when explicitly requested.

```yaml
services:
  app:
    build: .

  docs:
    image: mkdocs/mkdocs
    profiles: [dev]       # only started with --profile dev

  debug:
    image: busybox
    profiles: [dev]
```

```bash
docker compose --profile dev up
```

---

## PATTERNS

### DEV VS PROD OVERRIDES

```bash
# Base config + dev overrides
docker compose -f compose.yaml -f compose.dev.yaml up

# Base config + prod overrides
docker compose -f compose.yaml -f compose.prod.yaml up
```

### WAIT FOR A SERVICE

Use `depends_on` with `condition: service_healthy` and a `healthcheck` on the dependency (shown in the Compose file example above). Avoid `sleep` hacks.

### SECRETS (AVOID PLAINTEXT ENV)

```yaml
services:
  app:
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

The secret is mounted at `/run/secrets/db_password` inside the container.

---

## CLEANUP

```bash
# Remove everything unused (containers, images, networks, build cache)
docker system prune

# Include volumes
docker system prune --volumes

# Show disk usage
docker system df
```
