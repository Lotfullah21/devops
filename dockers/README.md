## What is Docker?

Docker a containerization platform that makes creating, deploying, and managing containers easy. Key features include:

## `Docker Images`:

Blueprints for containers, containing all the instructions for creating a container, they contains the following files.

- Runtime environment
- application code
- any dependencies the application needs to run
- configurations like environmental variables
- commands

images are read only, once created, cannot be edited later. to add any new information, a new images should be created.

When we run the image, it creates a container and container runs our application as outlined in the image.
containers runs in a completely isolated environment.

Docker images are made up of several layers and each layer is built on top of each other.

- `parent layer`:It contains the operating system and sometimes the run time environment.
- `source code`:
- `dependencies`:
- `run commands`:

## `Dockerfile`:

A text file that tells docker how to create an image.

- First, we need to add the version of parent layer
- Second, adding extra layers to copy the source code, where to add the, which commands to use and many other files.

- ### 1. Create a docker file without any extension named `Dockerfile`.
- ### 2. Start writing the lines, generally, each line represent a layer in our image
  - install `docker` by search `Docker` in extensions
- ### 3. add the first layer which will be the parent and something that can be downloaded from `dockerhub`
- ### 4. `FROM alpine:3.20`

  - `alpine` is the distribution of linux
  - `3.20`is the version

- ### 5. Copy all of the source code into an image
- `COPY . .`
  - the first `.` is the relative we want to copy the files from
  - the second `.` is the relative path to which the files should be copied
- Say, we want the copied folders to be added to a folder inside the image named `app`, then what shall be included in instead of second `.`/

  - `COPY . /app`

- ### 6. Install all the dependencies we need
  - `RUN pipenv install`, it install all the dependencies when building the container from the image
- ### 7. Add the `WORKDIR`.

  - this dir will be working directory where all commands will refer to this when doing some operation like installing the dependencies or copying the files.

- ### 7. Add the commands to be executed when running the application, it can be done using `CMD`.

  - `CMD ["python","manage.py"]`

- ### 8.Expose the port.

`Docker Engine`: The runtime that builds and runs containers

`Docker Hub`: A cloud repository where you can find and share container images
