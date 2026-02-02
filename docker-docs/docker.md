# Docker Essentials for Beginners

## Part 1 — What Is Docker?

Docker is a tool that packages your application and **all its dependencies** into a single unit called a **container**. That container runs the same way on your laptop, your teammate's machine, a test server, or production.

**The problem it solves:** "It works on my machine" — different OS versions, missing libraries, conflicting dependencies. Docker eliminates all of that by shipping the entire environment alongside your code.

**One-line mental model:** A container is a lightweight, isolated mini-computer running inside your real computer.

## Part 2 — Docker Under the Hood

Docker is not magic. It uses three Linux kernel features to create containers.

### 1. Namespaces — Isolation

Namespaces give each container its **own private view** of the system. The container thinks it's the only thing running.

| Namespace | What It Isolates                                                         |
| --------- | ------------------------------------------------------------------------ |
| **PID**   | Processes — container only sees its own processes, PID 1 is its main app |
| **NET**   | Network — container gets its own IP address, ports, routing table        |
| **MNT**   | Filesystem — container has its own root filesystem (`/`)                 |
| **UTS**   | Hostname — container can have its own hostname                           |
| **IPC**   | Inter-process communication — shared memory is isolated                  |
| **USER**  | User IDs — root inside the container can map to a non-root user outside  |

**Simple analogy:** Namespaces are like hotel rooms. Every guest (container) has their own room (PID, network, filesystem). They can't see into other rooms, even though they're all in the same building (host machine).

### 2. cgroups — Resource Limits

Control Groups (**cgroups**) limit **how much** of the host's resources a container can use.

| Resource     | What cgroups Control                                      |
| ------------ | --------------------------------------------------------- |
| **CPU**      | How much processor time the container gets                |
| **Memory**   | Maximum RAM (container is killed if it exceeds the limit) |
| **Disk I/O** | Read/write speed to disk                                  |
| **Network**  | Bandwidth limits                                          |

Without cgroups, one runaway container could eat all your server's memory and crash everything else. cgroups prevent that.

```bash
# Example: limit a container to 512MB RAM and 1 CPU core
docker run --memory=512m --cpus=1 myapp
```

### 3. Layers & Union Filesystem — Efficient Storage

Docker images are built in **layers**. Each layer is a read-only snapshot of filesystem changes.

```
┌─────────────────────────────┐
│ Layer 4: COPY app.py /app   │  <- your code (tiny)
├─────────────────────────────┤
│ Layer 3: RUN pip install    │  <- dependencies
├─────────────────────────────┤
│ Layer 2: WORKDIR /app       │  <- just metadata
├─────────────────────────────┤
│ Layer 1: FROM python:3.12   │  <- base OS + Python (shared)
└─────────────────────────────┘
```

**Why this matters:**

- **Caching** — If you change only your code (Layer 4), Docker reuses Layers 1–3 from cache. Builds are fast.
- **Sharing** — 10 containers from the same image share all read-only layers. Only a thin writable layer is unique per container.
- **Size** — Layers are deduplicated. If two images both use `python:3.12`, that base layer is stored only once on disk.

When a container writes a file, it goes into the **writable layer** on top. This is called **copy-on-write** — the original image layer is never modified.

### How It All Fits Together

```
You type: docker run myapp
              │
              ▼
   Docker Daemon (dockerd)
              │
              ▼
   containerd (manages lifecycle)
              │
              ▼
   runc (creates the container)
              │
              ├── Creates NAMESPACES (isolation)
              ├── Sets up CGROUPS (resource limits)
              ├── Mounts LAYERS (union filesystem)
              │
              ▼
   Your app is running in an isolated container
```

## Part 3 — Writing Dockerfiles from Scratch

A **Dockerfile** is a recipe — a plain text file that tells Docker how to build an image, step by step.

### Your First Dockerfile (Python App)

Let's say you have a simple `app.py`:

```python
# app.py
print("Hello from Docker!")
```

Create a file named `Dockerfile` (no extension):

```dockerfile
# Step 1: Start from a base image that has Python
FROM python:3.12-slim

# Step 2: Set the working directory inside the container
WORKDIR /app

# Step 3: Copy your code into the container
COPY app.py .

# Step 4: Define what happens when the container starts
CMD ["python", "app.py"]
```

Build and run it:

```bash
docker build -t my-first-app .
docker run my-first-app
# Output: Hello from Docker!
```

That's it. Four lines. Let's break down every instruction you'll actually use.

### The Core Instructions

#### `FROM` — Pick Your Starting Point

Every Dockerfile starts with `FROM`. It sets the base image.

```dockerfile
FROM python:3.12-slim    # Python with minimal OS
FROM node:20-alpine      # Node.js on tiny Alpine Linux
FROM ubuntu:22.04        # Full Ubuntu OS
FROM scratch             # Completely empty (for Go binaries)
```

**Tip:** Always use a specific tag (`python:3.12-slim`) not `python:latest`. Latest changes over time and will break your builds.

#### `WORKDIR` — Set the Directory

All subsequent commands run from this directory. Creates it if it doesn't exist.

```dockerfile
WORKDIR /app
```

#### `COPY` — Bring Files In

Copies files from your project into the image.

```dockerfile
COPY . .                           # copy everything
COPY package.json package-lock.json ./   # copy specific files
COPY src/ /app/src/                # copy a folder
```

#### `RUN` — Execute Commands During Build

Runs a command and saves the result as a new layer.

```dockerfile
RUN apt-get update && apt-get install -y curl    # install OS packages
RUN pip install -r requirements.txt              # install Python deps
RUN npm install                                  # install Node deps
```

**Important:** Each `RUN` creates a layer. Combine related commands with `&&` to keep images small:

```dockerfile
# Bad — 3 layers, leftover cache
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — 1 layer, clean
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

#### `CMD` — Default Startup Command

What runs when someone does `docker run your-image`. Only one CMD per Dockerfile (last one wins).

```dockerfile
CMD ["python", "app.py"]           # exec form (recommended)
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]
```

#### `EXPOSE` — Document the Port

Tells readers which port your app listens on. It does **not** actually publish the port — that's done with `docker run -p`.

```dockerfile
EXPOSE 3000
```

#### `ENV` — Set Environment Variables

Available both during build and at runtime.

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

#### `USER` — Don't Run as Root

For security, switch to a non-root user.

```dockerfile
RUN useradd -r appuser
USER appuser
```

### Real-World Examples

#### Python Flask App

```
my-project/
├── app.py
├── requirements.txt
├── Dockerfile
└── .dockerignore
```

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies first (cached if requirements.txt hasn't changed)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then copy app code (changes more often)
COPY . .

EXPOSE 5000

# Non-root user
RUN useradd -r appuser
USER appuser

CMD ["python", "app.py"]
```

#### Node.js Express App

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Dependencies first for caching
COPY package.json package-lock.json ./
RUN npm ci --only=production

# App code
COPY . .

EXPOSE 3000

USER node

CMD ["node", "server.js"]
```

#### Static Website with Nginx

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/
COPY css/ /usr/share/nginx/html/css/
COPY js/ /usr/share/nginx/html/js/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### The `.dockerignore` File

Just like `.gitignore`, this tells Docker what to **exclude** from the build context. Always create one.

```
node_modules
.git
.env
*.md
Dockerfile
docker-compose.yml
__pycache__
.venv
```

Without this, `docker build` sends your entire project directory (including `node_modules`, `.git`, secrets) to the daemon. Slow and potentially insecure.

### The Layer Caching Rule

Docker caches each layer. When rebuilding, it reuses cached layers **until it hits a line where something changed**. From that point on, everything rebuilds.

```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .        # <- changes rarely
RUN pip install -r requirements.txt  # <- cached if requirements.txt unchanged

COPY . .                       # <- changes often (your code)
CMD ["python", "app.py"]
```

**Golden rule:** Copy things that change **least** first, things that change **most** last.

If you put `COPY . .` before `RUN pip install`, every code change would re-install all dependencies. That's wasted time.

### Build, Tag, Run — The Workflow

```bash
# Build the image (. = use current directory as build context)
docker build -t myapp:v1.0 .

# See it in your local images
docker images

# Run it
docker run -d -p 3000:3000 --name my-container myapp:v1.0

# Check it's running
docker ps

# View logs
docker logs my-container

# Stop and remove
docker stop my-container
docker rm my-container
```

### Quick Reference Card

| Instruction  | What It Does                                 | When It Runs    |
| ------------ | -------------------------------------------- | --------------- |
| `FROM`       | Sets base image                              | Build           |
| `WORKDIR`    | Sets working directory                       | Build           |
| `COPY`       | Copies files into image                      | Build           |
| `ADD`        | Like COPY + URLs + tar extraction            | Build           |
| `RUN`        | Executes a command, saves result as layer    | Build           |
| `ENV`        | Sets environment variable                    | Build + Runtime |
| `ARG`        | Build-time variable only                     | Build           |
| `EXPOSE`     | Documents the port (no actual effect)        | Documentation   |
| `USER`       | Switches to a non-root user                  | Build + Runtime |
| `CMD`        | Default command when container starts        | Runtime         |
| `ENTRYPOINT` | Fixed executable (CMD becomes its arguments) | Runtime         |

## Key Takeaways

1. **Docker = Namespaces + cgroups + Layers.** Namespaces isolate, cgroups limit resources, layers make images efficient.
2. **A container is NOT a VM.** It shares the host kernel. No guest OS. That's why it's fast and lightweight.
3. **Images are read-only templates.** Containers add a thin writable layer on top.
4. **Dockerfiles are simple recipes.** `FROM` → `COPY` → `RUN` → `CMD`. That's the core pattern.
5. **Order matters for caching.** Put slow-changing things (dependencies) before fast-changing things (code).
6. **Always use `.dockerignore`.** Keep builds fast and secure.
7. **Never use `:latest` in production.** Pin your versions.
8. **Don't run as root.** Add a `USER` instruction.

_"Docker doesn't eliminate complexity — it packages it so you only deal with it once."_
