# Pipenv in Docker

- How Python dependencies are installed inside a container.

## What Is Pipenv?

Before we dive into the Docker commands, let's understand what Pipenv is.

When you build a Python project, you install packages — Django, Pillow, psycopg2, etc. You need a tool to manage those packages. That tool is **Pipenv**.

Pipenv combines two tools into one:

- **pip** — installs Python packages
- **virtualenv** — creates isolated environments so different projects don't conflict

Pipenv tracks your dependencies in two files:

| File           | What It Contains                                                                                                      |
| -------------- | --------------------------------------------------------------------------------------------------------------------- |
| `Pipfile`      | The packages you asked for (e.g., `django = "*"`)                                                                     |
| `Pipfile.lock` | The **exact** versions that were resolved and installed (e.g., `django==5.0.2`). Auto-generated. Never edit manually. |

```
    Pipfile              Pipfile.lock
    ┌────────────┐       ┌─────────────────────────┐
    │ django     │       │ django == 5.0.2         │
    │ pillow     │  ───► │ pillow == 10.2.0        │
    │ psycopg2   │       │ psycopg2 == 2.9.9      │
    │            │       │ + all sub-dependencies  │
    └────────────┘       └─────────────────────────┘
    "what I want"        "exactly what was installed"
```

`Pipfile.lock` is the important one. It guarantees that everyone — your teammate, the CI server, the Docker container — installs the **exact same versions**.

## Step 1 — Install Pipenv in the Container

```dockerfile
RUN pip install --no-cache-dir pipenv
```

This installs Pipenv itself inside the container so we can use it to install our project's dependencies.

### What Does `--no-cache-dir` Do?

When pip installs a package, it normally saves a copy in a cache folder so future installs are faster. Inside a Docker container, we'll never run this command again — the image is built once. So the cache is just wasted space.

```
    Without --no-cache-dir:
    ┌───────────────────────────┐
    │ pipenv installed    10 MB │
    │ pip cache           15 MB │  ← useless, never used again
    │ Total:              25 MB │
    └───────────────────────────┘

    With --no-cache-dir:
    ┌───────────────────────────┐
    │ pipenv installed    10 MB │
    │ pip cache            0 MB │  ← nothing wasted
    │ Total:              10 MB │
    └───────────────────────────┘
```

`--no-cache-dir` prevents caching of installation files, reducing the image size by avoiding unnecessary temporary data.

## Step 2 — Copy Dependency Files into the Container

```dockerfile
COPY Pipfile Pipfile.lock ./
```

### What Does COPY Do?

```
COPY source_on_your_machine destination_inside_container
```

It copies files from your local project directory into the container's current working directory (set by `WORKDIR`).

```
    Your Machine                        Container
    ┌───────────────┐                  ┌───────────────┐
    │ my-project/   │                  │ /app/         │
    │  ├── app.py   │     COPY        │                │
    │  ├── Pipfile ─┼────────────────►│  ├── Pipfile   │
    │  ├── Pipfile. │                  │  ├── Pipfile. │
    │  │   lock     ┼────────────────►│  │   lock      │
    │  └── ...      │                  │  └── (only    │
    └───────────────┘                  │      these)   │
                                       └───────────────┘
```

### Why Copy These Files First?

This command copies `Pipfile` and `Pipfile.lock` from your local project directory into the container's current working directory.

It prepares the container to install the exact same dependencies that were used in development, ensuring consistency across different environments. By copying these files, we ensure that our containerized application has precisely the same Python packages and versions as our local development environment.

We copy **only** the dependency files first (not the full source code) because of **layer caching**. If your code changes but your dependencies don't, Docker reuses the cached dependency layer and skips reinstalling packages. This makes builds much faster. (See [Layer Caching](./LAYER-CACHING.md) for a full explanation.)

```
    COPY Pipfile Pipfile.lock ./      cached (if deps didn't change)
    RUN pipenv install                cached (skips reinstall)
    COPY . .                          rebuilds (code changed)
```

## Step 3 — Install Python Dependencies

```dockerfile
RUN pipenv install --deploy --system
```

This is the command that actually installs all your Python packages inside the container. Let's break down each part:

### `pipenv install`

Manages Python package installation. Reads `Pipfile.lock` and installs every package listed in it.

### `--deploy`

Strictly enforces the **exact** package versions from `Pipfile.lock`.

What it does:

- Prevents installing packages not specified in the lock file
- **Fails the build** if `Pipfile.lock` is out of sync with `Pipfile`
- Ensures reproducible builds

```
    Without --deploy:
    Pipfile says django, Pipfile.lock is outdated
    → pipenv installs whatever version it finds    (unpredictable)

    With --deploy:
    Pipfile says django, Pipfile.lock is outdated
    → BUILD FAILS with an error                    (safe — forces you to fix it)
```

This is a safety net. If someone edited `Pipfile` but forgot to run `pipenv lock` to update `Pipfile.lock`, the build will catch the mistake instead of silently installing wrong versions.

### `--system`

Installs packages directly into the **global** Python environment instead of creating a virtual environment.

```
    Without --system (default pipenv behavior):
    ┌───────────────────────────────────┐
    │  Container                        │
    │  ┌─────────────────────────────┐  │
    │  │  Virtual Environment        │  │  ← unnecessary extra layer
    │  │  django, pillow, etc.       │  │
    │  └─────────────────────────────┘  │
    └───────────────────────────────────┘

    With --system:
    ┌───────────────────────────────────┐
    │  Container                        │
    │  django, pillow, etc.             │  ← installed directly
    │  (global Python environment)      │
    └───────────────────────────────────┘
```

Why skip the virtualenv? A virtual environment exists to **isolate** your project from other projects on the same machine. But a Docker container already **is** an isolated environment — there are no other projects in it. Creating a virtualenv inside a container is redundant. Using `--system` reduces overhead and simplifies dependency management.

## Key Benefits

| Benefit             | How It's Achieved                                                                                                                            |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Reproducibility** | `Pipfile.lock` + `--deploy` guarantees the same dependencies across all environments — your machine, your teammate's machine, CI, production |
| **Security**        | `--deploy` prevents unexpected package version variations that could introduce vulnerabilities                                               |
| **Performance**     | `--no-cache-dir` and `--system` reduce container image size and installation complexity                                                      |

## All Three Steps Together

```dockerfile
# 1. Install pipenv (the tool itself)
RUN pip install --no-cache-dir pipenv

# 2. Copy dependency files
COPY Pipfile Pipfile.lock ./

# 3. Install project dependencies
RUN pipenv install --deploy --system
```

```
    Step 1                Step 2                  Step 3
    Install the tool      Copy the recipe         Cook the recipe

    ┌──────────┐         ┌──────────────┐        ┌──────────────────┐
    │  pipenv   │         │  Pipfile     │        │  django          │
    │  binary   │         │  Pipfile.lock│───────►│  pillow          │
    │  installed│         │  copied in   │        │  psycopg2        │
    └──────────┘         └──────────────┘        │  + everything    │
                                                  │  from lock file  │
                                                  └──────────────────┘
```
