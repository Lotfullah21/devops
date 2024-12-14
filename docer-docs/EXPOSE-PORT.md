`## Expose 8000`

It informs docker that the container will listen to a specific port, but it it does not open or publish the port. By doing this, we inform or give hints to other developers what is the port number that this container is going to list to.

8000 is the default port for Django's development server

To actually make the port accessible, we'll need to use -p flag when running the container, like:

```sh
docker run -p 8000:8000 our-django-image
```

The `EXPOSE` instruction is a way of documenting and hinting at the container's network configuration without actually publishing the port.

# 0.0.0.0

This is a special `IP` address that tells the server to listen to all available network interface.

### Network interface

A network interface is a point of connection between a device and a network, enabling it to communicate with other devices.

### What it means in practice:

If our machine has multiple network interfaces (e.g., localhost, Wi-Fi, Ethernet), the server will be accessible from all of them.

### For example:

- `From our own machine`: we can access the server at http://localhost:8000 or http://127.0.0.1:8000.
- `From another machine on the same network`: we can access it using our machine's actual IP address, such as http://192.168.x.x:8000.

#### Why is this useful?

It makes the server more flexible, as it can be accessed from any network interface.
Developers often use `0.0.0.0` during development when they want the server to be reachable both locally and from other devices on the same network.

`listening`, means the server is actively waiting for incoming network connections on a port.

## 1. Physical vs. Virtual Network Interfaces

### Physical Network Interface

This is the hardware on our machine that connects to a network.

##### Examples:

- Ethernet port: For wired connections.
- Wi-Fi adapter: For wireless connections.

Each of these interfaces has its own unique IP address for identification.

### Virtual Network Interface

Software-based interfaces that simulate a network connection.

### Examples:

- Loopback interface (localhost/127.0.0.1): A virtual interface for testing, always present on every device.
- Docker network interfaces: Created by Docker to allow containers to communicate with the host and other containers.
- VPN adapters: Provide access to remote networks.
