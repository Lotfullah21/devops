## WORKDIR /app

Sets `/app` as the working directory inside the container. All subsequent commands will run relative to this directory.

all subsequent commands `(like RUN, CMD, COPY, etc.)` will execute relative to this directory, it helps us to organize our docker file.
Say, the `WORKDIR` is not present, then every time we have to mention where we do need to the operation.

```sh
# Without WORKDIR
RUN mkdir -p /app

COPY . /app

RUN cd /app && pipenv install --deploy --system
```

NOw, if `WORKDIR` is present,

```sh
# Set /app as the working directory
WORKDIR /app

# Copy application code
COPY . .

# Install Python dependencies
RUN pipenv install --deploy --system
```

Using WORKDIR simplifies and organizes our Dockerfile, and ensures that all commands will be executed in the expected directory inside the container.
