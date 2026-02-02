## Cloud SQL Proxy

A utility provided by google that allows secure, authenticated access to the database from local machine or other environments.

### How it works

```sh
./cloud-sql-proxy -p 5433 project-name
```

When we run the following commands, these things happens

### 1. Establishes a Secure Connection:

- The proxy uses our Google Cloud credentials (via gcloud auth) to authenticate with the Cloud SQL database.
- It establishes a secure, encrypted connection between our local machine and the Cloud SQL database.

### 2. Listens Locally:

- The proxy listens for incoming connections on a specific port (5433) on our local machine.
- When a program (like Django or psql) tries to connect to localhost:5433, the proxy intercepts that request.

### 3. Forwards Requests:

- After intercepting the request, the proxy securely forwards it to the Cloud SQL instance in Google Cloud.

### 4. Returns Responses:

- Responses from the Cloud SQL database are routed back through the proxy to the local application.

## Why Local Machine Can Access the Database

The proxy acts as a bridge between local machine and the Cloud SQL database. Here’s the flow:

#### Django App or psql Command:

- When you run your Django app (or use psql) and specify localhost:5433, your program is simply sending requests to the proxy running on your local machine.

#### Proxy Securely Communicates with Cloud SQL:

The proxy handles all the networking, authentication, and encryption required to communicate with the Cloud SQL database.

#### No Need for Direct Public Access:

Cloud SQL database remains private because the proxy communicates directly with it over Google's internal network, bypassing public internet exposure.
