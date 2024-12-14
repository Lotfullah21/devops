## PORT

A port is a communication end point used by network services on a computer.
With different ports, we can provide different services based on each port number.

## Examples:

Port 80: Default port for HTTP (web traffic).
Port 443: Default port for HTTPS (secure web traffic).
Port 5432: Default port for PostgreSQL.
Port 8000: Common port used in development servers like Django.

### What Does "Listening" Mean?

A server `listens` on a port by opening that port and waiting for incoming requests.
When a client (e.g., a browser, API call, or app) sends a request to the server's IP and port, the server processes the request and sends back a response.
The server does this continuously, waiting for new connections.

When a client sends data to that port, the operating system directs the data to the server process listening there.

### Listening in Practice

### Development Server:

When we run python manage.py runserver 0.0.0.0:8000, Django's development server binds to port 8000 and starts listening for requests.
We can access it via http://127.0.0.1:8000 (local access) or http://<our-ip>:8000 (if listening on 0.0.0.0).

### Database:

PostgreSQL might listen on port 5432, waiting for applications like Django to send database queries.

#### Production Server:

In a production setup, a web server like Nginx might listen on port 80 for HTTP traffic and forward requests to Django app.

#### Binding to a Port:

The server reserves the port for itself, ensuring no other application can use it.

#### Listening State:

The port is actively monitored for new connections.

#### Handling Connections:

Once a request arrives, the server processes it and sends back a response, then resumes listening for more requests.

# Network services

Network services are applications or processes that run on a networked computer and provide functionality to other devices over the network.

### Examples of Network Services

#### Web Servers:

Serve websites and web applications.
Examples: Apache, Nginx, or Django running on http://localhost:8000.

### Database Servers:

Provide access to databases for querying and data storage.
Examples: PostgreSQL, MySQL, MongoDB.

### Email Services:

Handle sending and receiving emails.
Examples: SMTP (Simple Mail Transfer Protocol) servers, IMAP (Internet Message Access Protocol) servers.

### Authentication Services:

Provide login, identity verification, and access control.
Examples: LDAP (Lightweight Directory Access Protocol), Firebase Authentication.

### DNS (Domain Name System):

Translates human-readable domain names (e.g., www.google.com) into IP addresses.

# How Network Services Work

### Server-Client Model:

A server provides the service (e.g., a web server delivering a webpage).
A client consumes the service (e.g., your browser requesting a webpage).

### Communication Protocols:

Services use protocols (rules for communication) like HTTP, FTP, SMTP, or TCP/IP.
Ports:

Each service "listens" on a specific port (e.g., port 80 for HTTP, port 443 for HTTPS) to distinguish itself from other services on the same machine.

### Importance of Network Services

##### Communication:

Enable users and devices to exchange data (e.g., email, file sharing, messaging).

##### Access to Resources:

Allow centralized resources like databases, websites, or files to be accessed by multiple devices.

##### Automation:

Power IoT devices, cloud computing, and APIs for automated workflows
