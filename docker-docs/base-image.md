# Base Image

## First — What Is a Docker Image?

An image is a **read-only package** that contains everything your app needs to run: the operating system, language runtime, libraries, your code, and the startup command. Think of it as a **snapshot** of a fully configured machine, frozen in time.

You don't run an image directly. You create a **container** from it — a running instance of that image.

```
┌──────────────────────────────────────────┐
│            CONTAINER                     │  <- Running instance (writable)
│   Your app is alive here                 │     Created with: docker run
├──────────────────────────────────────────┤
│            IMAGE                         │  <- Frozen snapshot (read-only)
│   OS + runtime + libraries + your code   │     Created with: docker build
├──────────────────────────────────────────┤
│         BASE IMAGE                       │  <- The foundation layer
│   Minimal OS + language runtime          │     Set with: FROM python:3.12
├──────────────────────────────────────────┤
│         HOST OS + DOCKER ENGINE          │  <- Your actual machine
│   Linux kernel (shared by all containers)│
└──────────────────────────────────────────┘
```

- **Host OS** — Your real machine. Docker runs on its Linux kernel.
- **Base Image** — The first layer. A minimal OS with just enough to run your app.
- **Image** — The base image + your dependencies + your code. Built by the Dockerfile.
- **Container** — A live, running copy of the image with a thin writable layer on top.

## What Is a Base Image?

The base image is the **first line** of your Dockerfile. It's the foundation everything else is built on.

```dockerfile
FROM python:3.12-slim
```

This gives you a minimal Linux OS with Python 3.12 pre-installed. Every instruction after this (`RUN`, `COPY`, `CMD`) adds layers on top.

## Choosing a Base Image

| Base Image           | Size   | Best For                                                               |
| -------------------- | ------ | ---------------------------------------------------------------------- |
| `alpine`             | ~5 MB  | Smallest possible. You install everything yourself.                    |
| `debian`             | ~22 MB | Standard Linux. More tools out of the box.                             |
| `ubuntu`             | ~29 MB | Popular, well-documented, general purpose.                             |
| `python:3.12-alpine` | ~25 MB | Lightweight Python. Can have issues compiling C extensions.            |
| `python:3.12-slim`   | ~45 MB | Recommended for Python apps. Debian-based, fewer compatibility issues. |
| `python:3.12`        | ~1 GB  | Full image. Has everything, but heavy. Avoid in production.            |

**For most Django/Python projects, use `python:3.12-slim`.** It balances size and compatibility — packages like `psycopg2` and `Pillow` compile without issues, unlike on Alpine where you often need extra workarounds.

## Example

```dockerfile
# This one line gives you: Linux OS + Python 3.12 + pip
FROM python:3.12-slim

# Everything below adds layers on top of it
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

## Quick Rule

- **Development / learning** → `python:3.12-slim` (easy, works out of the box)
- **Tiny production images** → `python:3.12-alpine` (small, but may need extra setup)
- **Go / Rust binaries** → `scratch` (completely empty, just your binary)
