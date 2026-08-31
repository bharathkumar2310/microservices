1. Container internals — next

Learn these in this order:

Container
↓
Namespaces
↓
cgroups
↓
Container filesystem
↓
Image layers
↓
Writable container layer

Focus especially on:

Namespaces → how containers are isolated
cgroups → how CPU/RAM are limited
Why containers don't have their own kernel
Why two containers can both have PID 1
How containers get their filesystem

You don't need deep Linux kernel knowledge yet.

2. Docker commands — then practice

Learn these practically:

docker pull
docker images
docker run
docker ps
docker ps -a
docker stop
docker start
docker restart
docker rm
docker logs
docker exec
docker inspect

For example:

docker run -d --name myapp nginx

Then understand:

docker run
↓
image
↓
container created
↓
container started

Then:

docker logs myapp
docker exec -it myapp sh
docker stop myapp
docker start myapp

Don't just memorize commands—understand what Docker is doing internally.

3. Port mapping

This is very important for microservices.

Learn:

docker run -p 8080:80 nginx

Understand:

Your Windows machine
localhost:8080
↓
Docker
↓
Container port 80
↓
Nginx

Then learn:

Container port
Host port
-p
Why localhost works
Basic Docker networking
4. Docker networking

Then learn:

Container A
│
│ Docker Network
▼
Container B

Understand:

bridge network
container-to-container communication
container IP
container DNS/name resolution
docker network ls
docker network create
docker network inspect

This is very important for microservices.

5. Docker volumes

Then:

Container
│
▼
Volume
│
▼
Persistent data

Understand:

Why container data can disappear
Volumes
Bind mounts
Why databases need persistent storage

Example:

docker run -v mysql-data:/var/lib/mysql mysql
6. Dockerfile

Only after the above, learn how an image is actually created.

Dockerfile
↓
docker build
↓
Docker Image
↓
docker run
↓
Container

Learn:

FROM
WORKDIR
COPY
RUN
EXPOSE
ENV
CMD
ENTRYPOINT

For your Java background, this is particularly important:

Java source
↓
Maven build
↓
JAR
↓
Dockerfile
↓
Docker image
↓
Container
7. Docker Compose

Then learn how multiple microservices run together:

docker-compose.yml

        ┌── Java service
        │
        ├── MySQL
        │
        ├── Redis
        │
        └── Kafka

Learn:

services
networks
volumes
environment variables
depends_on
docker compose up
docker compose down

This will make your microservices architecture much easier to understand.

8. Then Kubernetes

Only after all of that, move to Kubernetes:

Docker basics
↓
Docker networking
↓
Docker volumes
↓
Dockerfile
↓
Docker Compose
↓
Kubernetes

And then Kubernetes becomes much easier because you'll already understand what Kubernetes is managing.