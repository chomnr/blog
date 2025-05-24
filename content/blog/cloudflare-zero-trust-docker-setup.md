+++
title = "how to setup cloudflare zero trust using docker"
date = 2024-12-30
+++

This guide shows you how to route your server traffic through Cloudflare Zero Trust using Docker containers.

**What you'll need:**
- Server (optional)  
- Docker installed
- Cloudflare account

**Setting up your accounts:**

First, create a Cloudflare account at [https://dash.cloudflare.com/sign-up?pt=f](https://dash.cloudflare.com/sign-up?pt=f) and follow the setup steps. If you don't have Docker installed yet, follow the installation guide at [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/).

**Creating your Zero Trust tunnel:**

If you're new to Cloudflare and Docker, Zero Trust essentially creates a secure tunnel between your containers and Cloudflare's network without exposing ports directly to the internet.

Navigate to [https://dash.cloudflare.com/](https://dash.cloudflare.com/) (or go directly to [https://one.dash.cloudflare.com/](https://one.dash.cloudflare.com/)) and click on **Zero Trust**. Go to **Networks** → **Tunnels** and click **Add a tunnel**.

Follow the setup instructions, making sure to select **Cloudflared** instead of **WARP Connector**. When you reach the **Install and run connectors** step, select **Docker** as your environment.

Before running the connector command that Cloudflare provides, create a Docker network first:

```bash
docker network create tutorial
```

Now modify the Docker command from Cloudflare by adding the `--network tutorial` parameter. Your command should look like this:

```bash
docker run --network tutorial cloudflare/cloudflared:latest tunnel --no-autoupdate run --token secret_token_here
```

Run this command and your tunnel should connect successfully. Once connected, click on your tunnel name, select **Edit**, and go to **Public Hostname**. Choose your desired domain and path, set the type to **HTTP**, and for the URL use your Docker container's IP address with the appropriate port.

**Important:** Make sure any containers you want to route through the tunnel are on the same Docker network (`tutorial` in this example) as your tunnel container.