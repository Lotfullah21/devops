## Docker Commands

Here are some some common commands which are necessary for having a docker file.

## 1. Base Image

It specifies the base image for the container and it is the first layer in our image and it does not inherit from any other image.
It can also be said an image with minimal OS with the necessary libraries and packages pre-installed.

A base image is like the operating system of the container. It provides the fundamental environment required to run your application, such as the file system, libraries, and binaries.

### Popular Base Images:

`Alpine`: Lightweight and secure, ideal for reducing container size.
`Debian/Ubuntu`: Full-featured, widely used Linux distributions with many tools and libraries.
`Language-specific Images`: For example, python:3.10 includes Python 3.10 and essential libraries.

```py
FROM python:3.10-alpine
```

- This base image includes Python 3.10 and is built on top of Alpine Linux, a minimal OS image.
- The subsequent instructions in the Dockerfile add layers on top of this base image (e.g., installing packages, copying application code).

| **Base Image**       | **Size** | **Use Case**                                                                                  |
| -------------------- | -------- | --------------------------------------------------------------------------------------------- |
| `alpine`             | ~5 MB    | Minimal, for lightweight apps where you manually add only the dependencies you need.          |
| `debian`             | ~22 MB   | A more complete OS, useful when you need standard Linux tools or libraries.                   |
| `python:3.10-alpine` | ~25 MB   | Lightweight Python environment, ideal for Python applications with minimal dependencies.      |
| `python:3.10-slim`   | ~40 MB   | Slightly larger than Alpine, but easier to work with for applications needing more libraries. |
| `ubuntu`             | ~29 MB   | Popular, full-featured Linux OS image for general-purpose applications.                       |
