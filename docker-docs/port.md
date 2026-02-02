# Ports & Network Services

## What Is a Port?

A port is a **communication endpoint** used by network services on a computer. Think of an IP address as a building's street address, and a port as a specific **room number** inside that building. With different ports, we can provide different services on the same machine.

```
            Your Server (IP: 192.168.1.10)
    ┌──────────────────────────────────────────┐
    │                                          │
    │   Port 80  ──► Nginx (HTTP)              │
    │   Port 443 ──► Nginx (HTTPS)             │
    │   Port 5432──► PostgreSQL                │
    │   Port 8000──► Django Dev Server         │
    │                                          │
    └──────────────────────────────────────────┘

    One machine, multiple services, each on its own port.
```

### Common Port Numbers

| Port | Service    | Description                                    |
| ---- | ---------- | ---------------------------------------------- |
| 80   | HTTP       | Default port for web traffic                   |
| 443  | HTTPS      | Default port for secure web traffic            |
| 5432 | PostgreSQL | Default port for PostgreSQL database           |
| 3306 | MySQL      | Default port for MySQL database                |
| 6379 | Redis      | Default port for Redis cache                   |
| 22   | SSH        | Secure remote login                            |
| 8000 | Django Dev | Common port used in Django development servers |
| 8080 | Alt HTTP   | Common alternative HTTP port                   |

## What Does "Listening" Mean?

A server **listens** on a port by opening that port and waiting for incoming requests. When a client (e.g., a browser, API call, or app) sends a request to the server's IP and port, the server processes the request and sends back a response. The server does this continuously, waiting for new connections.

When a client sends data to that port, the operating system directs the data to the server process listening there.

```
    Client (Browser)                     Server
    ┌─────────────┐                   ┌─────────────┐
    │             │    request to     │             │
    │  Browser    │──────────────────►│  Port 8000  │
    │             │   port 8000      │  (Django)   │
    │             │◄──────────────────│             │
    │             │    response       │             │
    └─────────────┘                   └─────────────┘
```

### How Listening Works Step by Step

1. **Binding to a Port** — The server reserves the port for itself, ensuring no other application can use it.
2. **Listening State** — The port is actively monitored for new connections.
3. **Handling Connections** — Once a request arrives, the server processes it and sends back a response, then resumes listening for more requests.

### Listening in Practice

**Development Server:**
When we run `python manage.py runserver 0.0.0.0:8000`, Django's development server binds to port 8000 and starts listening for requests. We can access it via `http://127.0.0.1:8000` (local access) or `http://<our-ip>:8000` (if listening on `0.0.0.0`).

**Database:**
PostgreSQL might listen on port 5432, waiting for applications like Django to send database queries.

**Production Server:**
In a production setup, a web server like Nginx might listen on port 80 for HTTP traffic and forward requests to the Django app.

```
    Browser                  Nginx                Django
    ┌──────┐   port 80    ┌──────┐   port 8000  ┌──────┐
    │      │─────────────►│      │─────────────►│      │
    │      │◄─────────────│      │◄─────────────│      │
    └──────┘              └──────┘              └──────┘
                         (reverse proxy)       (app server)
```

## Network Services

Network services are applications or processes that run on a networked computer and provide functionality to other devices over the network.

### Examples of Network Services

**Web Servers:**
Serve websites and web applications. Examples: Apache, Nginx, or Django running on `http://localhost:8000`.

**Database Servers:**
Provide access to databases for querying and data storage. Examples: PostgreSQL, MySQL, MongoDB.

**Email Services:**
Handle sending and receiving emails. Examples: SMTP (Simple Mail Transfer Protocol) servers, IMAP (Internet Message Access Protocol) servers.

**Authentication Services:**
Provide login, identity verification, and access control. Examples: LDAP (Lightweight Directory Access Protocol), Firebase Authentication.

**DNS (Domain Name System):**
Translates human-readable domain names (e.g., `www.google.com`) into IP addresses.

## How Network Services Work

### Server-Client Model

A **server** provides the service (e.g., a web server delivering a webpage). A **client** consumes the service (e.g., your browser requesting a webpage).

```
    ┌──────────┐                    ┌──────────┐
    │  CLIENT  │    request         │  SERVER   │
    │          │───────────────────►│          │
    │ (browser,│                    │ (nginx,  │
    │  app,    │◄───────────────────│  django, │
    │  curl)   │    response        │  postgres)│
    └──────────┘                    └──────────┘
```

### Communication Protocols

Services use protocols (rules for communication) to exchange data:

| Protocol     | Purpose                               | Example                                  |
| ------------ | ------------------------------------- | ---------------------------------------- |
| HTTP / HTTPS | Web traffic                           | Browser loading a webpage                |
| TCP/IP       | General network communication         | Most internet traffic                    |
| FTP          | File transfer                         | Uploading files to a server              |
| SMTP         | Sending email                         | Your email client sending a message      |
| IMAP / POP3  | Receiving email                       | Your email client fetching messages      |
| DNS          | Domain name resolution                | Translating `google.com` → `142.250.x.x` |
| WebSocket    | Real-time bidirectional communication | Chat apps, live dashboards               |

### Ports

Each service "listens" on a specific port (e.g., port 80 for HTTP, port 443 for HTTPS) to distinguish itself from other services on the same machine.

## Importance of Network Services

**Communication:**
Enable users and devices to exchange data (e.g., email, file sharing, messaging).

**Access to Resources:**
Allow centralized resources like databases, websites, or files to be accessed by multiple devices.

**Automation:**
Power IoT devices, cloud computing, and APIs for automated workflows.

## How This Relates to Docker

In Docker, your container runs in its own isolated network. The app inside listens on a port (e.g., 8000), but that port is **only accessible inside the container** unless you explicitly map it to the host.

```dockerfile
# In Dockerfile — documents the port (does NOT open it)
EXPOSE 8000
```

```bash
# At runtime — actually maps host port to container port
docker run -p 8000:8000 my-django-app
#            ▲      ▲
#         host   container
```

- `EXPOSE 8000` in the Dockerfile is **documentation only**. It tells readers the app listens on 8000 but does not publish it.
- `-p 8000:8000` in `docker run` **actually maps** the port. Without this flag, the container's port is unreachable from your host.
- You can map to a different host port: `-p 5000:8000` means you visit `localhost:5000` but the container still listens on 8000 internally.
