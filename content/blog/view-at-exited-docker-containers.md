+++
title = "how to view exited docker containers"
date = 2025-06-05
+++

To view all Docker containers that have stopped running (exited status), you can use the following command:

```bash
docker ps -a --filter "status=exited"
```
