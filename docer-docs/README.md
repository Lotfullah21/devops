```sh
# Use a slim Python base image
FROM python:3.12-slim

# Set working directory inside the container
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libjpeg-dev \
    zlib1g-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install pipenv
RUN pip install --no-cache-dir pipenv

# Copy Pipfile and Pipfile.lock
COPY Pipfile Pipfile.lock ./

# Install Python dependencies
RUN pipenv install --deploy --system

# Copy the .env file into the container
COPY .env .env

# Copy application code
COPY . .

# Set environment variables for Django
ENV DJANGO_SETTINGS_MODULE=hooshmandlab.settings
ENV PYTHONUNBUFFERED=1

# Expose the port
EXPOSE 8000

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "hooshmandlab.wsgi:application"]
```

## Build the image

```sh
# build the image
docker build -t my-django-app .
# run the container
docker run -p 8000:8000 my-new-django-app
# to list all the running containers
docker ps
# to stop the running container with specific id
docker exec -it abc123def456 bash
```

## Useful commands

```sh
# show all the images
docker images
# create and run the container with a name from an image
docker run --name myapp_c1 myapp_image
# shows all the running container
docker ps
# shows all the container even if not running
docker ps -a
# stop the container with name or id
docker stop <container_name or id>
# access the container's port via 5000 in local host
# docker run --name myapp_c1 -p mapping_port:exposed_port -d(don't involve the terminal) myapp_image
docker run --name myapp_c1 -p 5000:4000 -d myapp_image
# start a container from existing containers
docker start <container_name or id>
```

- `docker build` creates a docker image from a docker file
- `-t my-django-app` option tags the resulting image the name `my-django-app.`
- `.` it tells the docker to use the contents of current directory to for the image build process

The build context is the set of files that Docker can access to include in the image. Docker will look at everything in the current directory and its subdirectories (including the Dockerfile and the .env file if they are in the current directory).

## Docs

- [docker](./DOCERK.md)
- [base-image](./BASE-IMAGE.md)
- [work-dir](./WORKDIR.md)
- [system](./SYSTEM.md)
- [pipenv](./PIPENV.md)
- [copy](./COPY.md)
- [cmd](./CMD.md)
- [django-settings](./DJANGO-SETTINGS.md)
- [expose](./EXPOSE-PORT.md)

<!-- Extra -->

- [port](./PORT.md)
- [layer-caching](./LAYER-CACHING.md)
