## Layer caching

Generally, each line in our docker file represents a new layer in our image and when we are building the image using `docker build -t image_name .` each layer is build on top of each other.
a docker image is a read-only file, meaning that once an image is created, it cannot be edited unless creating a brand new image. if we make even a slightest changed, do we have to download all the resources and repeat the steps?

```sh
# Use a slim Python base image
FROM python:3.12-slim

# Set working directory inside the container
WORKDIR /app

# Copy application code
COPY . .

# Install pipenv
RUN pip install  pipenv

# Install Python dependencies
RUN pipenv install --deploy --system

# Expose the port
EXPOSE 8000

# Command to run the app
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Let's say, we make a change in the source code, now all the steps after the `COPY . .` should be repeated again; why? since the layers are built on top of each other, layers after the source code needs to rebuilt again as the are dependent on previous layer.

Here is the problem, if we are making changes to source code, why do we have to install `pipenv` and then install all dependencies unless we make some changes in them?

### Why Layers Are Rebuilt

Each line in a Dockerfile creates a new layer in the image. Docker builds layers sequentially, and each layer depends on the previous ones. If any layer changes, Docker invalidates the cache for that layer and all subsequent layers, forcing them to rebuild.

What is the solution, how can we avoid this?

```sh
# use a slim Python base image
FROM python:3.12-slim

# set working directory inside the container
WORKDIR /app

# install pipenv
RUN pip install pipenv

# copy only Pipfile and Pipfile.lock first
COPY Pipfile Pipfile.lock ./

# install Python dependencies
RUN pipenv install --deploy --system

# copy application source code
COPY . .

# Expose the port
EXPOSE 8000

# Command to run the app
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

before copying the source code, install `pipenv`, copy `pipfiles` and install the `packages` and then copy the source code. Now, when building an image, we don't need to build the layers from scratch, directly those unchanged layers can be picked from the cached files.

### General Best Practices for Layer Caching

- Minimize Layers: Combine commands like RUN apt-get update && apt-get install -y ... && rm -rf /var/lib/apt/lists/\* to reduce layers.
- Order Layers Strategically: Put the least frequently changed layers (e.g., dependency installation) before frequently changing layers (e.g., source code).
- Separate Build and Runtime: Use multi-stage builds to separate dependency installation from the runtime environment, reducing the final image size.
- Use .dockerignore: Exclude unnecessary files (e.g., logs, temporary files) from the build context to prevent invalidating the cache.

## Which Instructions Create Layers?

The following lines create layers:

FROM: Base layer (always creates one).
WORKDIR /app: Creates a new layer.
COPY . .: Creates a new layer (dependent on source code).
RUN pipenv install --deploy --system: Creates a new layer.
RUN pip install pipenv: Creates a new layer.
Non-layer-Creating Instructions:
EXPOSE: Metadata, no layer created.
CMD: Metadata for runtime, no layer created.

When combined, --deploy and --system ensure that:

```sh
# Install Python dependencies
RUN pipenv install --deploy --system
```

dependencies are installed exactly as specified in Pipfile.lock `(--deploy)`.
packages are installed directly into the container's global environment `(--system)`.

## What is the Build Context in Docker?

The build context in Docker refers to the set of files and directories that are accessible to the Docker daemon during the build process. It consists of everything within the directory (and its subdirectories) where we run the docker build command, except for files or directories excluded by a .dockerignore file.

## How Does Build Context Work?

When we run docker build -t <image-name> ., the . at the end specifies the current directory as the build context. Docker transfers the build context to the Docker daemon, which uses it to execute the instructions in the Dockerfile.
