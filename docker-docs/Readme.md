# Dockerizing a Django Application

A step-by-step guide to containerizing a Django project with Pipenv and Gunicorn.

## Docs

| Topic                                   | Description                                    |
| --------------------------------------- | ---------------------------------------------- |
| [Docker Overview](./docker.md)          | What Docker is and how it works                |
| [Base Image](./base-image.md)           | Choosing the right `FROM` image                |
| [Working Directory](./workdir.md)       | Setting `WORKDIR` inside the container         |
| [System Dependencies](./system.md)      | Installing OS-level packages with `apt-get`    |
| [Pipenv](./pipenv.md)                   | Managing Python dependencies with Pipenv       |
| [Copy](./copy.md)                       | How `COPY` works and what to copy when         |
| [CMD](./cmd.md)                         | The startup command for your container         |
| [Django Settings](./django-settings.md) | Configuring Django inside Docker               |
| [Expose](./expose-port.md)              | What `EXPOSE` does (and doesn't do)            |
| [Port Mapping](./port.md)               | Mapping container ports to your host           |
| [Layer Caching](./layer-caching.md)     | How Docker caches layers and why order matters |

## The Dockerfile

```dockerfile
# ── 1. Base Image ────────────────────────────────────────────
# Start from a minimal Python image (~150MB vs ~1GB for full)
FROM python:3.12-slim

# ── 2. Working Directory ─────────────────────────────────────
# All subsequent commands run from /app
WORKDIR /app

# ── 3. System Dependencies ───────────────────────────────────
# Install OS packages needed to compile Python packages
# Combined into one RUN to keep the image small (single layer)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libjpeg-dev \
    zlib1g-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ── 4. Install Pipenv ───────────────────────────────────────
RUN pip install --no-cache-dir pipenv

# ── 5. Copy Dependency Files First (Layer Caching) ──────────
# These change rarely → Docker caches this layer
COPY Pipfile Pipfile.lock ./

# ── 6. Install Python Dependencies ──────────────────────────
# --deploy: fail if Pipfile.lock is out of date
# --system: install globally (no virtualenv inside container)
RUN pipenv install --deploy --system

# ── 7. Copy Application Code ────────────────────────────────
# This changes often → placed AFTER dependency install
COPY . .

# ── 8. Django Configuration ─────────────────────────────────
ENV DJANGO_SETTINGS_MODULE=hooshmandlab.settings
ENV PYTHONUNBUFFERED=1

# ── 9. Expose Port ──────────────────────────────────────────
# Documents that the app listens on 8000 (does NOT publish it)
EXPOSE 8000

# ── 10. Startup Command ─────────────────────────────────────
# Gunicorn serves the Django WSGI application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "hooshmandlab.wsgi:application"]
```

### What Each System Package Does

| Package           | Why It's Needed                                                              |
| ----------------- | ---------------------------------------------------------------------------- |
| `build-essential` | C compiler and tools — required to compile Python packages with C extensions |
| `libpq-dev`       | PostgreSQL client headers — required by `psycopg2`                           |
| `libjpeg-dev`     | JPEG library — required by `Pillow` for image handling                       |
| `zlib1g-dev`      | Compression library — required by `Pillow`                                   |

### Why `--no-install-recommends`?

Skips optional suggested packages. Keeps the image smaller by only installing exactly what's needed.

### Why `apt-get clean && rm -rf /var/lib/apt/lists/*`?

Removes the package cache in the **same layer** as the install. If this were a separate `RUN`, the cache would still exist in the previous layer and the image wouldn't actually shrink.

## .dockerignore

Create this file in your project root to keep the build context clean and prevent sensitive files from leaking into the image:

```
.git
.gitignore
.env
*.pyc
__pycache__
.venv
*.md
Dockerfile
docker-compose*.yml
.DS_Store
.vscode
.idea
node_modules
```

- **Security note:** Avoid copying `.env` into the image with `COPY .env .env`. Secrets baked into images are visible via `docker history`. Instead, pass them at runtime with `docker run --env-file .env` or use Docker secrets.

## Build & Run

### Build the Image

```bash
docker build -t my-django-app .
```

| Part               | Meaning                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `docker build`     | Reads the Dockerfile and builds an image                                                                            |
| `-t my-django-app` | Tags (names) the image `my-django-app`                                                                              |
| `.`                | **Build context** — the directory Docker can access. It sends everything here (minus `.dockerignore`) to the daemon |

### Run the Container

```bash
docker run -d -p 8000:8000 --name django_app my-django-app
```

| Part                | Meaning                                                     |
| ------------------- | ----------------------------------------------------------- |
| `-d`                | Detached mode — runs in the background, frees your terminal |
| `-p 8000:8000`      | Maps host port 8000 → container port 8000                   |
| `--name django_app` | Gives the container a human-readable name                   |
| `my-django-app`     | The image to create the container from                      |

Your app is now live at **http://localhost:8000**.

### Run with Environment Variables (Recommended)

```bash
docker run -d -p 8000:8000 --name django_app --env-file .env my-django-app
```

This passes your `.env` file at **runtime** instead of baking it into the image.

## Essential Commands

### Images

```bash
# List all local images
docker images

# Build an image
docker build -t my-django-app .

# Remove an image
docker rmi my-django-app

# Remove all unused images
docker image prune -a
```

### Containers

```bash
# Create and run a new container
docker run -d -p 8000:8000 --name django_app my-django-app

# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Stop a running container
docker stop django_app

# Start a stopped container
docker start django_app

# Restart a container
docker restart django_app

# Remove a stopped container
docker rm django_app

# Force-remove a running container
docker rm -f django_app
```

### Debugging

```bash
# Open a shell inside the running container
docker exec -it django_app bash

# View logs
docker logs django_app

# Follow logs in real-time
docker logs -f django_app

# View last 50 lines
docker logs --tail 50 django_app

# Check resource usage
docker stats django_app

# Inspect full container metadata
docker inspect django_app
```

### Cleanup

```bash
# Remove all stopped containers
docker container prune

# Remove all unused images, containers, networks
docker system prune

# Nuclear option — remove everything unused including volumes
docker system prune -a --volumes
```

## Port Mapping Explained

```
         HOST                    CONTAINER
    ┌─────────────┐         ┌─────────────┐
    │              │         │             │
    │  port 8000 ──┼────────►── port 8000 │
    │              │   -p    │  (EXPOSE)   │
    │  browser     │         │  gunicorn   │
    │              │         │             │
    └─────────────┘         └─────────────┘

    -p HOST_PORT:CONTAINER_PORT
    -p 8000:8000
```

- `EXPOSE 8000` in the Dockerfile is **documentation only**. It tells readers the app listens on 8000 but does not open anything.
- `-p 8000:8000` in `docker run` **actually maps** the port. Without this flag, the container's port is unreachable from your host.
- You can map to a different host port: `-p 5000:8000` means you visit `localhost:5000` but the container still listens on 8000 internally.

## Layer Caching — Why Order Matters

Docker caches each layer. When rebuilding, it reuses cached layers **until it hits a changed line**. Everything after that rebuilds from scratch.

```
                    Dockerfile Order
                    ────────────────
  Changes rarely    FROM python:3.12-slim            cached
                    WORKDIR /app                     cached
                    RUN apt-get install ...          cached
                    RUN pip install pipenv           cached
                    COPY Pipfile Pipfile.lock ./     cached (if unchanged)
                    RUN pipenv install               cached (if deps unchanged)
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
  Changes often     COPY . .                        rebuilds (code changed)
                    CMD ["gunicorn", ...]           rebuilds
```

**This is why we copy `Pipfile` before `COPY . .`** — a code change won't trigger a full dependency reinstall. Only if `Pipfile` or `Pipfile.lock` change will pip packages be reinstalled.

## Common Mistakes to Avoid

| Mistake                             | Why It's Bad                                                         | Fix                                                  |
| ----------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------- |
| `COPY .env .env` in Dockerfile      | Secrets are baked into the image and visible via `docker history`    | Use `--env-file .env` at runtime                     |
| No `.dockerignore`                  | Sends `.git`, `__pycache__`, `.venv` to the build — slow and bloated | Create a `.dockerignore` file                        |
| `COPY . .` before `RUN pip install` | Every code change reinstalls all dependencies                        | Copy dependency files first, install, then copy code |
| Using `FROM python:3.12` (full)     | ~1GB image with unnecessary tools                                    | Use `python:3.12-slim` (~150MB)                      |
| Running as root                     | Security risk in production                                          | Add `RUN useradd -r appuser` and `USER appuser`      |
| Using `:latest` tag                 | Builds break when the upstream image changes                         | Pin versions: `python:3.12-slim`                     |
