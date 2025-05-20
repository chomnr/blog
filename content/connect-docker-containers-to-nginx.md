+++
title = "Tutorial: Connecting Docker Containers to NGINX"
date = 2025-01-05
+++

# How to Connect Your Docker Containers to NGINX

## Prerequisites

- Server (Optional)
- Docker
- NGINX

## NGINX Installation

You can set up NGINX **via or without Docker**, but this example will use Docker.

Refer to the official NGINX installation guide:  
[https://nginx.org/en/docs/install.html](https://nginx.org/en/docs/install.html)

## Docker Installation

Before doing anything, install Docker first.  
Refer to the Docker documentation for installation instructions:  
[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

## Setup

Assuming you already have NGINX set up, you should mount the `nginx.conf` on your server if you're using Docker.  
If you don't know how to mount files in Docker, see this guide:  
[https://www.baeldung.com/ops/docker-mount-single-file-in-volume](https://www.baeldung.com/ops/docker-mount-single-file-in-volume)

1. **Create a Docker network**

Create a network that the NGINX container and your other containers will share.  
Your containers and NGINX **must** share the same network.

```bash
docker network create %networkname%
```

2. Now that the network is created, you must assign the container the network with the flag below.

```bash
bash --network %networkname%
```

3.  Then run the command below and make sure that the containers are part of the same network.

```bash
network inspect %networkname%
```

4. Now go into your nginx.conf and use the following server block code to resolve your desired Docker container.

```conf
server {
    listen              80;
    server_name         example.com;

    resolver 127.0.0.11 valid=10s;
    resolver_timeout 5s;

    set $backend_service http://container_name:container_port;

    location / {
       proxy_pass $backend_service;

       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

5. This SHOULD resolve. If it still doesn't, the container and NGINX are not part of the same network, or the container doesn't have a port exposed.

DNS resolution only works on custom networks. If you're hosting NGINX on the actual machine instead of a container, DNS resolution via the container's name might not work. If not, try using the container's IP instead.
