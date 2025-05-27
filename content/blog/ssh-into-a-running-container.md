+++
title = "how to ssh into a running container"
date = 2025-05-26
+++

To connect to a running Docker container and get a shell inside it, you can use the `docker exec` command. Here's the basic usage:

```bash
docker exec -it <container-name> /bin/bash