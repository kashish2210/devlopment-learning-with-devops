# Docker Commands — Notes

## Images

**List all local images**
```bash
docker images
```

**Delete an image**
```bash
docker rmi <image>
```

**Remove unused images**
```bash
docker image prune
```

**Build an image from a Dockerfile**
```bash
docker build -t <name>:<tag> .        # tag/version is optional
docker build -t <name>:<tag> . --no-cache   # build without cache
```

---

## Containers

**List all local containers (running & stopped)**
```bash
docker ps -a
```

**List all running containers**
```bash
docker ps
```

**Create & run a new container**
```bash
docker run <image>
```
> If the image isn't available locally, it'll be downloaded from Docker Hub.

**Run container in background (detached)**
```bash
docker run -d <image>
```

**Run container with a custom name**
```bash
docker run --name <container-name> <image>
```

**Port binding**
```bash
docker run -p <host-port>:<container-port> <image>
```

**Set environment variables in a container**
```bash
docker run -e <VAR_NAME>=<value> <image>
```

**Start/stop/remove a container**
```bash
docker start <container>
docker stop <container>
docker rm <container>
```

**Get an interactive shell inside a running container**
```bash
docker exec -it <container> /bin/bash
docker exec -it <container> sh        # fallback if bash isn't available
```

**View container logs**
```bash
docker logs <container>
docker logs -f <container>            # follow (stream live)
```

---

## Docker Hub

**Pull an image from Docker Hub**
```bash
docker pull <image>
```

**Push an image to Docker Hub**
```bash
docker push <username>/<image>
```

**Login to Docker Hub**
```bash
docker login -u <username>
# or
docker login
```

**Logout (remove stored credentials)**
```bash
docker logout
```

**Search for an image on Docker Hub**
```bash
docker search <term>
```

---

## Volumes

**List all volumes**
```bash
docker volume ls
```

**Create a new named volume**
```bash
docker volume create <volume-name>
```

**Delete a named volume**
```bash
docker volume rm <volume-name>
```

**Mount a named volume with a running container**
```bash
docker run --volume <volume-name>:<container-path> <image>
# or using --mount
docker run --mount type=volume,src=<volume-name>,dst=<container-path> <image>
```

**Mount an anonymous volume with a running container**
```bash
docker run --volume <container-path> <image>
```

**Create a bind mount** (maps a host directory into the container)
```bash
docker run --volume <host-path>:<container-path> <image>
# or using --mount
docker run --mount type=bind,src=<host-path>,dst=<container-path> <image>
```

**Remove unused local volumes** (anonymous volumes not attached to any container)
```bash
docker volume prune
```

---

## Networks

**List all networks**
```bash
docker network ls
```

**Create a network**
```bash
docker network create <network-name>
```

**Remove a network**
```bash
docker network rm <network-name>
```

**Remove all unused networks**
```bash
docker network prune
```

**Attach a running container to a network**
```bash
docker network connect <network-name> <container>
```

**Inspect a network** (see connected containers, subnet, etc.)
```bash
docker network inspect <network-name>
```

---

## Quick Concept Notes

- **Named volume** → managed by Docker, persists independently of any single container, referenced by name (e.g. `jenkins-data`).
- **Anonymous volume** → created automatically without a name when only a container path is given; harder to reference later, cleaned up with `docker volume prune`.
- **Bind mount** → maps a specific path on the host machine directly into the container; useful for local development where you want live file syncing.
- Containers on the **same custom network** can resolve each other by **container name** via Docker's built-in DNS. This does **not** work on the default bridge network — only on user-created networks (`docker network create ...`).