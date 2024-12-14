1. ## Install pipenv

```sh
RUN pip install --no-cache-dir pipenv
```

## Purpose:

Installs pipenv, a dependency manager that combines pip and virtualenv. The `--no-cache-dir` flag ensures no unnecessary cache files are stored, reducing image size.

`pipenv`: a tool that combines `pip` and `virtualenv` fo r

#### `--no-cache-dir Flag`:

Prevents caching of installation files, reducing the image size by avoiding unnecessary temporary data.

2. ## COPY

```sh
COPY source__data_to_be_copied destination_of_copied_file_to_be_saved.
```

```sh
COPY pipfile pipfile.lock .
```

This command copies these files from local project directory into the container's current working directory.

It prepares the container to install the exact same dependencies that were used in development, ensuring consistency across different environments.

By copying these files, we ensure that our containerized application has precisely the same Python packages and versions as our local development environment.

## 3. Installation

```sh
RUN pipenv install --deploy --system
```

`pipenv install`: Manages Python package installation

`--deploy`: strictly enforces the exact package versions from Pipfile.lock

- Prevents installing packages not specified in the lock file
- Ensures reproducible builds

`--system`: Installs packages directly into the global Python environment

- Skips creating a separate virtual environment
- Reduces overhead in container environments
- Simplifies dependency management

### Key Benefits:

`Reproducibility`: Guarantees the same dependencies across all environments
`Security`: Prevents unexpected package version variations
`Performance`: Reduces container image size and installation complexity
