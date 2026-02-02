# Nginx, Gunicorn & Web Servers

Understanding how your Django app actually serves requests in production.

## What Is a Server?

A server is just a **computer that provides services to other computers**. When someone says "server," they usually mean one of two things:

**Hardware server** — A physical (or virtual) machine sitting in a data center, always powered on, always connected to the internet.

**Software server** — A program running on that machine that **listens** on a port and **responds** to incoming requests.

When we talk about Nginx or Gunicorn, we mean software servers.

```
    You (Client)                          Server Machine
    ┌──────────┐                       ┌──────────────────┐
    │          │   "show me the       │                  │
    │ Browser  │   homepage please"   │  Software Server │
    │          │─────────────────────►│  (listening on   │
    │          │                       │   port 80)       │
    │          │◄─────────────────────│                  │
    │          │   "here's the HTML"  │                  │
    └──────────┘                       └──────────────────┘
```

### Types of Software Servers

| Type                   | What It Does                                                                    | Examples                   |
| ---------------------- | ------------------------------------------------------------------------------- | -------------------------- |
| **Web Server**         | Serves static files (HTML, CSS, JS, images) and forwards dynamic requests       | Nginx, Apache              |
| **Application Server** | Runs your application code (Python, Node, etc.) and generates dynamic responses | Gunicorn, uWSGI, Daphne    |
| **Database Server**    | Stores and retrieves data                                                       | PostgreSQL, MySQL, MongoDB |
| **Mail Server**        | Sends and receives emails                                                       | Postfix, SendGrid          |

In a Django production setup, you typically use **two** of these together: a web server (Nginx) and an application server (Gunicorn).

## What Is Gunicorn?

Gunicorn (Green Unicorn) is a **Python application server**. Its only job is to run your Django code.

Django ships with `manage.py runserver` for development, but that server is:

- Single-threaded (handles one request at a time)
- Not secure (no SSL, no rate limiting)
- Not optimized (slow under load)

Gunicorn replaces it in production. It spawns **multiple worker processes**, each capable of handling requests independently.

```
                         Gunicorn
                    ┌──────────────────┐
    Request ───────►│  Master Process  │
                    │                  │
                    │  ┌────────────┐  │
                    │  │ Worker 1   │──┼──► runs Django code
                    │  └────────────┘  │
                    │  ┌────────────┐  │
                    │  │ Worker 2   │──┼──► runs Django code
                    │  └────────────┘  │
                    │  ┌────────────┐  │
                    │  │ Worker 3   │──┼──► runs Django code
                    │  └────────────┘  │
                    └──────────────────┘

    Multiple workers = multiple requests handled simultaneously
```

### How You Run Gunicorn

```bash
gunicorn --bind 0.0.0.0:8000 --workers 3 hooshmandlab.wsgi:application
```

| Part                            | Meaning                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `--bind 0.0.0.0:8000`           | Listen on port 8000, accept connections from any IP            |
| `--workers 3`                   | Spawn 3 worker processes to handle requests in parallel        |
| `hooshmandlab.wsgi:application` | The WSGI entry point — tells Gunicorn where your Django app is |

### What Gunicorn Does NOT Do

Gunicorn is great at running Python code, but it is **not designed** to:

- Serve static files efficiently (CSS, JS, images)
- Handle SSL/HTTPS termination
- Load balance across multiple servers
- Compress responses
- Cache content
- Rate limit or block abusive clients

That's where Nginx comes in.

## What Is Nginx?

Nginx (pronounced **"engine-x"**) is a high-performance **web server** and **reverse proxy**. It sits in front of Gunicorn and handles everything that Gunicorn shouldn't.

Think of it this way:

- Nginx = the receptionist at a hotel front desk
- Gunicorn = the staff who actually do the work in the back

The receptionist:

- Greets every guest (accepts all incoming connections)
- Answers simple questions directly (serves static files)
- Routes complex requests to the right department (forwards to Gunicorn)
- Handles security (SSL, rate limiting)
- Manages the queue so staff aren't overwhelmed (load balancing)

## How Nginx and Gunicorn Work Together

### The Full Request Flow

```
                    THE PRODUCTION STACK

    Browser                Nginx               Gunicorn             Django
    ┌──────┐            ┌──────┐            ┌──────────┐        ┌──────────┐
    │      │  request   │      │  forward   │          │  call  │          │
    │ User │───────────►│ :80  │───────────►│  :8000   │───────►│  views   │
    │      │            │      │            │          │        │  models  │
    │      │◄───────────│      │◄───────────│          │◄───────│  logic   │
    │      │  response  │      │  response  │          │  HTML  │          │
    └──────┘            └──────┘            └──────────┘        └──────────┘

    Port 80              Port 80 -> 8000       Port 8000           Internal
    (internet)           (reverse proxy)      (app server)        (your code)
```

### Step by Step

1. **User types `https://yoursite.com`** — the browser sends a request to port 443 (HTTPS) or 80 (HTTP).
2. **Nginx receives the request** — it's the first thing the outside world talks to.
3. **Nginx checks: is this a static file?**
   - If YES (CSS, JS, image) -> Nginx serves it directly from disk. Fast. Gunicorn never knows about it.
   - If NO (a Django URL like `/api/users`) -> Nginx forwards it to Gunicorn on port 8000.
4. **Gunicorn receives the forwarded request** — passes it to a worker process running Django.
5. **Django processes the request** — runs your view, queries the database, renders the template.
6. **The response flows back** — Django -> Gunicorn -> Nginx -> Browser.

```
    Request: GET /static/css/style.css

    Browser ──► Nginx ──► Serves directly from /static/ folder ──► Browser
                          (Gunicorn is never involved)

    ─────────────────────────────────────────────────────────────

    Request: GET /api/users/

    Browser ──► Nginx ──► Gunicorn ──► Django (views.py) ──► Database
                                                  │
    Browser ◄── Nginx ◄── Gunicorn ◄── Django ◄───┘
```

## What Nginx Does — The Complete Picture

### 1. Reverse Proxy

Nginx sits between the internet and your app server. The outside world only talks to Nginx — they never see Gunicorn directly.

```
    WITHOUT reverse proxy (dangerous):

    Internet ──► Gunicorn :8000 ──► Django
                 (exposed directly to the internet)

    WITH reverse proxy (safe):

    Internet ──► Nginx :80 ──► Gunicorn :8000 ──► Django
                 (only Nginx is exposed)
```

**Why this matters:** Gunicorn isn't built to handle raw internet traffic — slow clients, malicious requests, connection floods. Nginx is hardened for this and shields Gunicorn from the chaos.

### 2. Serving Static Files

Django can serve static files with `python manage.py runserver`, but it's extremely slow at it. Nginx serves files **directly from disk** without involving Python at all.

```nginx
# Nginx configuration
server {
    listen 80;

    # Static files — served directly by Nginx
    location /static/ {
        alias /app/staticfiles/;
    }

    # Media uploads — served directly by Nginx
    location /media/ {
        alias /app/media/;
    }

    # Everything else — forward to Gunicorn
    location / {
        proxy_pass http://gunicorn:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```
    /static/css/style.css    ->  Nginx handles it    (instant, no Python)
    /media/uploads/photo.jpg ->  Nginx handles it    (instant, no Python)
    /api/users/              ->  Nginx -> Gunicorn    (Python processes it)
    /admin/                  ->  Nginx -> Gunicorn    (Python processes it)
```

### 3. SSL/HTTPS Termination

Nginx handles the encryption/decryption of HTTPS traffic. Gunicorn and Django only deal with plain HTTP internally.

```
    Browser                    Nginx                     Gunicorn
    ┌──────┐   HTTPS (443)   ┌──────┐   HTTP (8000)    ┌──────┐
    │      │════════════════►│      │──────────────────►│      │
    │      │   encrypted     │      │   plain HTTP      │      │
    │      │◄════════════════│      │◄──────────────────│      │
    └──────┘                 └──────┘                    └──────┘

    ═══ = encrypted (HTTPS)
    ─── = plain (HTTP, internal network only)
```

Nginx holds the SSL certificate and does all the heavy encryption work. Your Django app doesn't need to know anything about SSL.

### 4. Load Balancing

When your app grows, you run multiple Gunicorn instances. Nginx distributes requests across them.

```
                                    ┌──────────────────┐
                              ┌────►│ Gunicorn :8001   │
                              │     └──────────────────┘
    Browser ──► Nginx :80 ────┤     ┌──────────────────┐
                              ├────►│ Gunicorn :8002   │
                              │     └──────────────────┘
                              │     ┌──────────────────┐
                              └────►│ Gunicorn :8003   │
                                    └──────────────────┘

    Nginx decides which instance handles each request.
    If one goes down, Nginx stops sending traffic to it.
```

```nginx
# Nginx load balancer config
upstream django_app {
    server gunicorn1:8000;
    server gunicorn2:8000;
    server gunicorn3:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://django_app;
    }
}
```

### 5. Response Compression

Nginx compresses responses (gzip) before sending them to the browser, reducing bandwidth and speeding up page loads.

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
```

```
    Without compression:  style.css = 150 KB
    With gzip:            style.css = 25 KB   (6x smaller)
```

### 6. Rate Limiting & Security

Nginx can block abusive clients, limit request rates, and restrict access.

```nginx
# Limit to 10 requests per second per IP
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20;
        proxy_pass http://gunicorn:8000;
    }
}
```

### 7. Caching

Nginx can cache responses so repeated requests don't hit Gunicorn at all.

```
    First request:   Browser -> Nginx -> Gunicorn -> Django -> DB
                                 │
                                 └── Nginx caches the response

    Second request:  Browser -> Nginx -> Returns cached response
                              (Gunicorn never touched)
```

## Summary — Who Does What

| Task                                 | Nginx | Gunicorn | Django              |
| ------------------------------------ | ----- | -------- | ------------------- |
| Accept internet traffic              | Yes   | No       | No                  |
| Serve static files (CSS, JS, images) | Yes   | No       | No (slow, dev only) |
| SSL/HTTPS encryption                 | Yes   | No       | No                  |
| Load balancing                       | Yes   | No       | No                  |
| Response compression (gzip)          | Yes   | No       | No                  |
| Rate limiting / security             | Yes   | No       | No                  |
| Caching                              | Yes   | No       | No                  |
| Run multiple Python workers          | No    | Yes      | No                  |
| Execute Python/Django code           | No    | No       | Yes                 |
| Handle business logic                | No    | No       | Yes                 |
| Query the database                   | No    | No       | Yes                 |
| Render templates                     | No    | No       | Yes                 |

## What This Looks Like in Docker Compose

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80" # only Nginx is exposed to the internet
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - static-data:/app/staticfiles
    depends_on:
      - web

  web:
    build: .
    command: gunicorn --bind 0.0.0.0:8000 --workers 3 hooshmandlab.wsgi:application
    volumes:
      - static-data:/app/staticfiles
    expose:
      - "8000" # internal only — not accessible from outside
    depends_on:
      - db

  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  db-data:
  static-data:
```

```
    Internet ──► Nginx (:80) ──► Gunicorn (:8000) ──► Django ──► PostgreSQL
                   │
                   └── serves /static/ and /media/ directly
```

Notice: only Nginx has `ports:` (exposed to the host). Gunicorn uses `expose:` (internal only). The database has no port mapping at all. The outside world only ever talks to Nginx.

## Development vs Production

|                  | Development                     | Production                |
| ---------------- | ------------------------------- | ------------------------- |
| **Web server**   | None (Django dev server)        | Nginx                     |
| **App server**   | `manage.py runserver`           | Gunicorn                  |
| **Static files** | Django serves them (slow)       | Nginx serves them (fast)  |
| **HTTPS**        | Not needed (`http://localhost`) | Nginx handles SSL         |
| **Workers**      | Single thread                   | Multiple Gunicorn workers |
| **Port exposed** | 8000 directly                   | 80/443 via Nginx          |
