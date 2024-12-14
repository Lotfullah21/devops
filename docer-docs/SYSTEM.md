```sh
# Install system dependencies required for Python and Django
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libjpeg-dev \
    zlib1g-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### Purpose:

### Installs essential system-level dependencies:

- build-essential: Required for compiling Python packages with C extensions.
- libpq-dev: Required for PostgreSQL database support.
- libjpeg-dev and zlib1g-dev: Required for image handling (e.g., Pillow library).
- apt-get clean and rm -rf /var/lib/apt/lists/\*: Clean up temporary files to reduce the final image size.

## purpose

Django application can successfully install required Python packages (e.g., those needing C extensions).
It can interact with a PostgreSQL database.
It can handle image processing tasks (like resizing or compression).
The resulting Docker image is optimized for production with no unnecessary bloat.

### Why These Specific Dependencies Are Required:

#### **`build-essential`**

- This is a meta-package that includes GCC (GNU Compiler Collection), `make`, and other tools required to compile software.
- Some Python packages (like `psycopg2` for PostgreSQL or `Pillow` for image handling) need to compile C code during installation.
- Without this, the `pip install` step for such packages would fail.

#### **`libpq-dev`**

- Provides header files and libraries for PostgreSQL, allowing Python libraries like `psycopg2` to interact with a PostgreSQL database.
- Essential for Django applications using PostgreSQL as their database backend.

#### **`libjpeg-dev` and `zlib1g-dev`**

- Required for the `Pillow` Python library, which handles image processing in Django (e.g., resizing images, handling profile pictures, etc.).
- **`libjpeg-dev`**: Provides JPEG image support.
- **`zlib1g-dev`**: Provides compression support for image files (e.g., PNG).
