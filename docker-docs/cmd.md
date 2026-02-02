## CMD

```sh
# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "your_project.wsgi:application"]
```

### `CMD`:

- The Docker instruction to specify the command that will be executed when the container starts
- Default command to run the application.

### `gunicorn`:

- Replaces Django's built-in development server
- Handles multiple concurrent requests efficiently
- Supports process management and worker configuration

### Network Binding (0.0.0.0:8000)

- Tells a server which network interface and port to listen on
- Determines how and where the application receives network connections

### Network Interface

0.0.0.0: Listens on ALL available network interfaces
127.0.0.1: Listens only on localhost (local machine)
Specific IP addresses can also be used

## Port Number

- Specifies which communication endpoint to use
- In web applications, typically 80 (HTTP) or 8000, 8080

### Practical Examples

### 0.0.0.0:8000:

- Listen on ALL network interfaces
- Accessible from external machines
- Typical for web servers and containerized applications

### 127.0.0.1:8000:

- Only accessible from the local machine
- Used for local development
- Cannot be reached from other computers

### Why 0.0.0.0 Matters

- Enables external access to your application

### Crucial for:

- Docker containers
- Cloud deployments
- Network services

Would you like me to elaborate on any of these points?

### WSGI(Web server gateway interface) Application Reference

- A standard interface between web servers and Python web applications
- Defines how a web server communicates with Python web applications
- Allows different web servers to work with different Python web frameworks

`your_project.wsgi`:application points to the Django project's WSGI configuration
Tells Gunicorn exactly how to load and run the Django application
Typically references the wsgi.py file in your Django project directory
