+++
title = "Setup Cloudflare Zero Trust with Docker"
date = 2024-12-30
+++

# Prerequisites

- Server (Optional)
- Docker
- Cloudflare

## Cloudflare Installation

Setting up Cloudflare is incredibly simple.  
Create an account here: [https://dash.cloudflare.com/sign-up?pt=f](https://dash.cloudflare.com/sign-up?pt=f)  
Follow the steps accordingly.

## Docker Installation

Before we can do anything, we have to install Docker first.  
To install Docker, please refer to the Docker documentation:  
[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

## Zero Trust

If you're unfamiliar with Cloudflare and Docker, this might be unclear, but I'll try to explain it as simply as possible.

1. Go to [https://dash.cloudflare.com/](https://dash.cloudflare.com/)  
   Alternatively, go directly to [https://one.dash.cloudflare.com/](https://one.dash.cloudflare.com/)

2. Click on **Zero Trust**

3. Click on **Networks** → **Tunnels**

4. Click on the **Create a tunnel** button

5. Follow the instructions and ensure you select **Cloudflared**, not **WARP Connector**

6. Once you reach **Install and run connectors**, select **Docker** as the environment

7. Before installing and running the connector, create a Docker network using the following command:

   ```bash
   docker network create tutorial
   ```

   You will now need to add the --network parameter to the command that was provided to you by Cloudflare. Now the command will look like this

   ```bash
   docker run --network tutorial cloudflare/cloudflared:latest tunnel --no-autoupdate run --token secret_token_here
   ```

   If you've done everything correctly, your tunnel should be connected to your container.

8. Now click on the tunnel you just created, click edit, go to Public Hostname, and choose the domain and path you would like your container to be connected to. For type, ensure you choose \*HTTP, and for URL, use the IP of the docker container /w port.

Please note that the desired container must be part of the same container as the tunnel container.
