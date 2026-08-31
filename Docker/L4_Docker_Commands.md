# Docker Commands — Easy Practical Notes

> Format: **2–3 line explanation → command → example → important options**

---

## 1. `docker --version`

This checks whether Docker is installed and shows the Docker CLI version.  
Use it as the first quick check when working with Docker.

```bash
docker --version
```

Example:

```text
Docker version 28.x.x
```

---

## 2. `docker info`

This gives detailed information about the Docker environment.  
It is useful when troubleshooting Docker because it shows information about the Docker server, containers, images, storage, CPU, memory, and runtime.

```bash
docker info
```

---

# Images

## 3. `docker pull`

`docker pull` downloads an image from a container registry to your local Docker environment.  
You use it when you want an image available locally before creating a container.

```bash
docker pull nginx
```

Specific version:

```bash
docker pull mysql:8.0
```

Here:

```text
mysql → image name
8.0   → tag/version
```

---

## 4. `docker images`

`docker images` lists the Docker images currently available on your machine.  
It helps you check whether the image you want to use is already downloaded.

```bash
docker images
```

You can also use:

```bash
docker image ls
```

---

## 5. `docker image inspect`

This gives detailed information about an image, including its configuration and metadata.  
It is useful when you want to understand exactly what an image contains/configures.

```bash
docker image inspect nginx
```

---

## 6. `docker history`

This shows the image's build history/layers.  
It helps you see how an image was built and which layers contributed to it.

```bash
docker history nginx
```

---

## 7. `docker rmi`

`docker rmi` removes an image from your local Docker environment.  
The image generally cannot be removed while it is still required by existing containers.

```bash
docker rmi nginx
```

---

# Containers

## 8. `docker run`

`docker run` creates a **new container from an image and starts it**.  
If the image is not available locally, Docker can pull it first.

```bash
docker run nginx
```

Think:

```text
Image
  ↓
Create NEW container
  ↓
Start container
```

---

## 9. `docker run -d`

`-d` means **detached mode**, so the container runs in the background instead of occupying your terminal.  
This is commonly used for servers such as nginx, MySQL, Redis, and Java applications.

```bash
docker run -d nginx
```

---

## 10. `--name`

`--name` gives the container a name that is easier to remember than its generated name or ID.  
You can then use that name with commands such as `docker stop`, `docker logs`, and `docker exec`.

```bash
docker run -d --name my-nginx nginx
```

Now:

```text
Container name = my-nginx
Image          = nginx
```

---

## 11. `docker ps`

`docker ps` shows the containers that are **currently running**.  
It is one of the first commands to use when checking whether your container is up.

```bash
docker ps
```

---

## 12. `docker ps -a`

`docker ps -a` shows **all containers**, including stopped containers.  
This is useful when you know a container existed but it is no longer running.

```bash
docker ps -a
```

Remember:

```text
docker ps
    ↓
running containers

docker ps -a
    ↓
all containers
```

---

## 13. `docker start`

`docker start` starts an **existing stopped container**.  
It does not create a new container.

```bash
docker start my-nginx
```

Important:

```text
docker run   = create + start NEW container
docker start = start EXISTING container
```

---

## 14. `docker stop`

`docker stop` stops a running container but does not delete it.  
You can start the same container again later.

```bash
docker stop my-nginx
```

Then:

```bash
docker start my-nginx
```

Think:

```text
stop = turn it off
```

---

## 15. `docker restart`

`docker restart` restarts an existing container.  
Conceptually, it stops the container and starts it again.

```bash
docker restart my-nginx
```

---

## 16. `docker rm`

`docker rm` removes a container.  
Stopping a container does not delete it; `rm` is the command used when you want to remove the container itself.

```bash
docker stop my-nginx
docker rm my-nginx
```

Force removal:

```bash
docker rm -f my-nginx
```

Remember:

```text
stop = container still exists
rm   = container is deleted
```

---

# Debugging Containers

## 17. `docker logs`

`docker logs` shows the output produced by the application running inside a container.  
This is extremely useful for debugging Java/Spring Boot applications.

```bash
docker logs my-app
```

To continuously watch new logs:

```bash
docker logs -f my-app
```

Here:

```text
-f = follow
```

---

## 18. `docker exec`

`docker exec` lets you execute a command inside an already-running container.  
It is useful when you want to inspect files, processes, environment variables, or configuration from inside the container.

```bash
docker exec -it my-nginx sh
```

Here:

```text
-i = interactive
-t = terminal
```

Inside the container:

```bash
ls
pwd
ps
env
```

Exit:

```bash
exit
```

If the image has Bash:

```bash
docker exec -it my-nginx bash
```

---

## 19. `docker inspect`

`docker inspect` gives detailed information about a container.  
It is very useful for troubleshooting configuration, networking, mounts, environment variables, image information, and container state.

```bash
docker inspect my-nginx
```

---

## 20. `docker stats`

`docker stats` shows live resource usage of running containers.  
You can use it to see CPU, memory, network I/O, and block I/O usage.

```bash
docker stats
```

For one container:

```bash
docker stats my-nginx
```

---

## 21. `docker top`

`docker top` shows processes running inside a container from Docker's perspective.  
It is useful when checking what processes the container is running.

```bash
docker top my-nginx
```

---

# Port Mapping

## 22. `-p`

`-p` maps a port on your computer to a port inside the container.  
For example, nginx can listen on port 80 inside the container while you access it through port 8080 on your computer.

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

Format:

```text
-p HOST_PORT:CONTAINER_PORT
```

So:

```text
Your computer
localhost:8080
      ↓
    Docker
      ↓
Container:80
      ↓
    nginx
```

Open:

```text
http://localhost:8080
```

---

# Environment Variables

## 23. `-e`

`-e` passes an environment variable into the container.  
This is commonly used for application configuration such as Spring profiles, database URLs, usernames, and other settings.

```bash
docker run -d   --name myapp   -e SPRING_PROFILES_ACTIVE=prod   my-java-app
```

Inside the container:

```text
SPRING_PROFILES_ACTIVE=prod
```

Check it:

```bash
docker exec myapp env
```

---

# Networking

## 24. `docker network ls`

This lists the Docker networks available on your machine.  
Networks allow containers to communicate with each other in a controlled way.

```bash
docker network ls
```

---

## 25. `docker network create`

This creates a custom Docker network.  
You can place multiple containers on the same network so they can communicate with each other.

```bash
docker network create my-network
```

---

## 26. `--network`

`--network` starts a container connected to a specific Docker network.  
Containers on the same network can communicate with each other.

```bash
docker run -d   --name app   --network my-network   nginx
```

Another container:

```bash
docker run -d   --name redis   --network my-network   redis
```

Conceptually:

```text
             my-network
              /       \
             /         \
           app         redis
```

---

## 27. `docker network inspect`

This shows detailed information about a Docker network.  
It is useful for checking which containers are connected and understanding the network configuration.

```bash
docker network inspect my-network
```

---

# Volumes

## 28. `docker volume ls`

This lists Docker-managed volumes.  
Volumes are commonly used to keep data separate from the container lifecycle.

```bash
docker volume ls
```

---

## 29. `docker volume create`

This creates a named Docker volume.  
Named volumes are commonly used for persistent data such as database files.

```bash
docker volume create mysql-data
```

---

## 30. `-v`

`-v` mounts a volume or host directory into a container.  
For databases, this lets data survive even if the container itself is removed.

```bash
docker run -d   --name mysql   -v mysql-data:/var/lib/mysql   mysql:8.0
```

Conceptually:

```text
MySQL Container
      ↓
mysql-data volume
      ↓
Persistent data
```

---

## 31. `docker volume inspect`

This shows detailed information about a Docker volume.  
It is useful when checking where the volume is stored and how it is configured.

```bash
docker volume inspect mysql-data
```

---

## 32. `docker volume rm`

This removes a Docker volume.  
Be careful: removing a volume can remove the persistent data stored in it.

```bash
docker volume rm mysql-data
```

---

# Copy Files

## 33. `docker cp`

`docker cp` copies files between your host machine and a container.  
It can copy in either direction.

Host → container:

```bash
docker cp app.jar my-container:/app/app.jar
```

Container → host:

```bash
docker cp my-container:/app/log.txt .
```

Think:

```text
Host ↔ Container
```

---

# Cleanup

## 34. `docker system prune`

This removes unused Docker resources such as stopped containers and other resources depending on what is unused.  
Use it carefully because cleanup commands can delete resources you may still want.

```bash
docker system prune
```

---

# Most Important Commands to Practice First

Do not try to memorize everything at once.

Start with:

```bash
docker --version

docker pull nginx

docker images

docker run -d --name my-nginx nginx

docker ps

docker ps -a

docker logs my-nginx

docker exec -it my-nginx sh

docker stop my-nginx

docker start my-nginx

docker restart my-nginx

docker inspect my-nginx

docker rm my-nginx
```

Then learn these options:

```text
-p          → port mapping
-e          → environment variable
-v          → volume/mount
--network   → connect container to a Docker network
-d          → background/detached mode
--name      → container name
```

---

# Complete Beginner Practice

## Step 1 — Pull nginx

```bash
docker pull nginx
```

## Step 2 — Check the image

```bash
docker images
```

## Step 3 — Run a container

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

## Step 4 — Check it

```bash
docker ps
```

## Step 5 — Open nginx

```text
http://localhost:8080
```

## Step 6 — Check logs

```bash
docker logs my-nginx
```

## Step 7 — Enter the container

```bash
docker exec -it my-nginx sh
```

Try:

```bash
ls
pwd
ps
env
```

Then:

```bash
exit
```

## Step 8 — Stop it

```bash
docker stop my-nginx
```

## Step 9 — Compare

```bash
docker ps
docker ps -a
```

Notice that the stopped container appears in `docker ps -a`.

## Step 10 — Start it again

```bash
docker start my-nginx
```

## Step 11 — Remove it

```bash
docker stop my-nginx
docker rm my-nginx
```

---

# Docker Learning Path

You already understand:

```text
Image
Container
Docker daemon
containerd
runc
```

Continue in this order:

```text
Docker Commands              ← CURRENT
       ↓
Port Mapping
       ↓
Docker Networking
       ↓
Volumes
       ↓
Dockerfile
       ↓
Build a Java Image
       ↓
Docker Compose
       ↓
Namespaces + cgroups
       ↓
Kubernetes
```

The goal is not to memorize commands. Understand **what each command does to the image/container**.
