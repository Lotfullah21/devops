# Layer Caching

## How Layers Work

Generally, each line in our Dockerfile represents a new layer in our image. When we build the image using `docker build -t image_name .`, each layer is built on top of each other.

A Docker image is a **read-only** file, meaning that once an image is created, it cannot be edited unless creating a brand new image. So if we make even the slightest change, do we have to download all the resources and repeat the steps?

No — Docker **caches** each layer. On the next build, it reuses cached layers until it hits one that changed. From that point on, everything rebuilds.

```
     Dockerfile                          Image Layers
     ──────────                          ────────────
     FROM python:3.12-slim       -->   ┌──────────────────┐
     WORKDIR /app                -->   │  Layer 1: base   │
     RUN pip install pipenv      -->   │  Layer 2: workdir│
     COPY Pipfile* ./            -->   │  Layer 3: pipenv │
     RUN pipenv install          -->   │  Layer 4: files  │
     COPY . .                    -->   │  Layer 5: deps   │
                                      │  Layer 6: code   │
                                      └──────────────────┘
                                      EXPOSE, CMD = metadata only
```

## The Problem — Bad Layer Order

Look at this Dockerfile:

```dockerfile
# Use a slim Python base image
FROM python:3.12-slim

# Set working directory inside the container
WORKDIR /app

# Copy application code
COPY . .

# Install pipenv
RUN pip install pipenv

# Install Python dependencies
RUN pipenv install --deploy --system

# Expose the port
EXPOSE 8000

# Command to run the app
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Let's say we make a change in the source code. Now all the steps after `COPY . .` have to be repeated again. Why? Since the layers are built on top of each other, layers after the source code need to be rebuilt because they are dependent on the previous layer.

Here is the problem: if we are only making changes to source code, why do we have to install `pipenv` and then install all dependencies again, unless we actually made changes to them?

```
FROM python:3.12-slim          cached
WORKDIR /app                   cached
COPY . .                       CHANGED — your code is different
RUN pip install pipenv         rebuilds (because previous layer changed)
RUN pipenv install             rebuilds (reinstalls ALL packages)
EXPOSE 8000                    metadata
CMD [...]                      metadata
```

**A tiny code change reinstalls every single dependency.** Slow and wasteful.

### Why Layers Are Rebuilt

Each line in a Dockerfile creates a new layer in the image. Docker builds layers sequentially, and each layer depends on the previous ones. If any layer changes, Docker **invalidates the cache** for that layer and all subsequent layers, forcing them to rebuild.

## The Solution — Correct Layer Order

What is the solution? How can we avoid this?

Before copying the source code, install `pipenv`, copy the `Pipfiles`, and install the packages — **then** copy the source code. Now when building an image, we don't need to build the layers from scratch. Those unchanged layers can be picked directly from the cached files.

```dockerfile
# Use a slim Python base image
FROM python:3.12-slim

# Set working directory inside the container
WORKDIR /app

# Install pipenv
RUN pip install pipenv

# Copy only Pipfile and Pipfile.lock first
COPY Pipfile Pipfile.lock ./

# Install Python dependencies
RUN pipenv install --deploy --system

# Copy application source code
COPY . .

# Expose the port
EXPOSE 8000

# Command to run the app
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Now the same code change:

```
FROM python:3.12-slim           cached
WORKDIR /app                    cached
RUN pip install pipenv          cached
COPY Pipfile Pipfile.lock ./    cached (files didn't change)
RUN pipenv install              cached (deps didn't change)
COPY . .                       CHANGED — only this rebuilds
EXPOSE 8000                    metadata
CMD [...]                      metadata
```

**Dependencies are served from cache. Build takes seconds instead of minutes.**

## The Rule — Order By Change Frequency

```
Things that change LEAST    -->   top of Dockerfile      (cached)
Things that change MOST     -->   bottom of Dockerfile   (rebuilt)

 ┌──────────────────────────────────────────────┐
 │  FROM python:3.12-slim          rarely       │
 │  WORKDIR /app                   changes      │
 │  RUN pip install pipenv                      │
 │  COPY Pipfile Pipfile.lock ./                │
 │  RUN pipenv install                          │
 ├──────────────────────────────────────────────┤
 │  COPY . .                       changes      │
 │  CMD [...]                      often        │
 └──────────────────────────────────────────────┘
```

## Which Instructions Create Layers?

Not every line creates a layer. Only instructions that **modify the filesystem** do.

**Layer-creating instructions:**

| Instruction                            | Creates a Layer? | Why                                                     |
| -------------------------------------- | ---------------- | ------------------------------------------------------- |
| `FROM`                                 | Yes              | Base layer (always creates one)                         |
| `WORKDIR /app`                         | Yes              | Creates/sets a new directory                            |
| `COPY . .`                             | Yes              | Adds files to the filesystem (dependent on source code) |
| `RUN pip install pipenv`               | Yes              | Executes a command, modifies the filesystem             |
| `RUN pipenv install --deploy --system` | Yes              | Installs packages, modifies the filesystem              |

**Non-layer-creating instructions (metadata only):**

| Instruction  | Creates a Layer? | Why                                                       |
| ------------ | ---------------- | --------------------------------------------------------- |
| `EXPOSE`     | No               | Metadata — documents the port, no filesystem change       |
| `CMD`        | No               | Metadata — sets the runtime command, no filesystem change |
| `ENV`        | No               | Metadata — sets environment variables                     |
| `ENTRYPOINT` | No               | Metadata — sets the executable                            |

---

## Pipenv Flags Explained

```dockerfile
# Install Python dependencies
RUN pipenv install --deploy --system
```

When combined, `--deploy` and `--system` ensure that:

| Flag       | What It Does                                                                                                                                                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deploy` | Dependencies are installed **exactly** as specified in `Pipfile.lock`. The build **fails** if `Pipfile.lock` is out of sync with `Pipfile`. This ensures reproducible installs.                                                    |
| `--system` | Packages are installed directly into the **container's global Python environment** instead of creating a virtualenv. Inside a container you already _are_ the isolated environment — a virtualenv inside a container is redundant. |

## General Best Practices for Layer Caching

- **Minimize Layers:** Combine commands like `RUN apt-get update && apt-get install -y ... && rm -rf /var/lib/apt/lists/*` to reduce layers.
- **Order Layers Strategically:** Put the least frequently changed layers (e.g., dependency installation) before frequently changing layers (e.g., source code).
- **Separate Build and Runtime:** Use multi-stage builds to separate dependency installation from the runtime environment, reducing the final image size.
- **Use `.dockerignore`:** Exclude unnecessary files (e.g., logs, temporary files) from the build context to prevent invalidating the cache.

## What Is the Build Context in Docker?

The build context in Docker refers to the set of files and directories that are accessible to the Docker daemon during the build process. It consists of everything within the directory (and its subdirectories) where we run the `docker build` command, except for files or directories excluded by a `.dockerignore` file.

### How Does Build Context Work?

When we run `docker build -t <image-name> .`, the `.` at the end specifies the current directory as the build context. Docker transfers the build context to the Docker daemon, which uses it to execute the instructions in the Dockerfile.

```
docker build -t myapp .
                       │
                       └── "send everything in this directory to the Docker daemon"
```

Docker transfers the **entire directory** (and subdirectories) to the daemon. This is why a `.dockerignore` matters — without it, Docker sends `.git`, `__pycache__`, `.venv`, and everything else, making builds slow and potentially leaking sensitive files.

```
# .dockerignore
.git
__pycache__
.venv
*.pyc
.env
node_modules
```

## Summary

1. Each line in a Dockerfile creates a layer. Layers are cached and reused on subsequent builds.
2. If a layer changes, Docker invalidates the cache for that layer **and all subsequent layers**.
3. Put things that change **rarely** (dependencies) before things that change **often** (source code).
4. Copy dependency files (`Pipfile`, `Pipfile.lock`) separately, install, then copy the rest of the code.
5. Use `.dockerignore` to keep the build context small, fast, and secure.
