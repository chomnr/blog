+++
title = "How to Connect Docker Containers to NGINX"
date = 2025-01-05
+++

This guide shows you how to connect your Docker containers to NGINX for reverse proxying and load balancing.

**What you'll need:**
- Server (optional)
- Docker installed 
- NGINX

**Getting started:**

You can set up NGINX with or without Docker, but this example uses Docker. If you need to install NGINX, check the official guide at [https://nginx.org/en/docs/install.html](https://nginx.org/en/docs/install.html). For Docker installation, follow the documentation at [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/).

If you're running NGINX in Docker, you'll need to mount your `nginx.conf` file from your server. If you're unfamiliar with mounting files in Docker, this guide explains it well: [https://www.baeldung.com/ops/docker-mount-single-file-in-volume](https://www.baeldung.com/ops/docker-mount-single-file-in-volume).

**Setting up the network:**

The key to connecting containers to NGINX is using a shared Docker network. Create a network that both NGINX and your application containers will use:

```bash
docker network create %networkname%
```

When running your containers, add the network flag to ensure they can communicate:

```bash
--network %networkname%
```

Verify that your containers are on the same network by inspecting it:

```bash
docker network inspect %networkname%
```

**Configuring NGINX:**

Add this server block to your `nginx.conf` to proxy requests to your Docker container:

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

Replace `container_name` with your actual container name and `container_port` with the port your application is listening on inside the container.

**Troubleshooting:**

If the connection isn't working, check that both containers are on the same network and that your application container has the correct port exposed. DNS resolution using container names only works on custom networks - if you're running NGINX directly on the host machine instead of in a container, you might need to use the container's IP address instead of its name.