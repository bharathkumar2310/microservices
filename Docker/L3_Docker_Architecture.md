# Docker Architecture — Easy-to-Understand Notes

> Goal: Understand **why Docker has Docker Engine, dockerd, containerd, runc, Linux Kernel, Docker Desktop, WSL2, and Kubernetes** without memorizing confusing definitions.

---

# 1. First Understand the Big Picture

Imagine you have a Java application.

Without Docker, your server needs things like:

```text
Server
│
├── Operating System
├── Java
├── Libraries
├── Configuration
└── Your Java Application
```

The problem is that another server may have a different Java version or different libraries.

For example:

```text
Your laptop:
Java 21
Application works

Production:
Java 17
Application fails
```

Docker helps us package the application and its required **user-space software** together.

```text
Docker Image
│
├── Your Application
├── Java Runtime
├── Libraries
├── Configuration
└── Required Files
```

That image can then be used to create containers.

---

# 2. Image vs Container

This is the first thing you should understand.

## Image = Blueprint

An image is a **read-only package/template**.

Example:

```text
my-java-app:1.0
```

Conceptually:

```text
Image
│
├── Java
├── Application JAR
├── Dependencies
└── Required filesystem
```

## Container = Running Instance

When you run:

```bash
docker run my-java-app:1.0
```

Docker creates a container from the image.

Think of it like:

```text
Image = Blueprint
Container = House built from the blueprint
```

One image can create many containers:

```text
              Java Image
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Container A Container B Container C
```

You do **not** need a completely separate image for every container.

---

# 3. Important Question: Does Every Container Have Its Own Linux OS?

## No.

This is one of the most important Docker concepts.

Suppose you run:

```text
Java Container
Kafka Container
Redis Container
```

It is NOT:

```text
Java → Linux OS
Kafka → Linux OS
Redis → Linux OS
```

Instead, conceptually:

```text
             Linux Kernel
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Java      Kafka      Redis
    Container  Container  Container
```

The containers share the underlying Linux kernel.

Each container has its own isolated environment and filesystem, but normally **not its own kernel**.

---

# 4. Then Why Does a Container Look Like Linux?

Suppose your image is based on Ubuntu.

The container may contain:

```text
Container
│
├── /bin
├── /etc
├── /usr
├── /var
├── Java
└── Your application
```

So when you enter the container, it looks like a small Linux system.

But remember:

```text
Linux filesystem ≠ Linux kernel
```

The container can contain Linux user-space files and programs.

The Linux kernel comes from the underlying Linux environment.

---

# 5. VM vs Container

This difference makes Docker much easier to understand.

## Virtual Machine

A VM normally has its own operating system and kernel.

```text
Physical Hardware
      │
      ▼
   Hypervisor
      │
 ┌────┴────┐
 ▼         ▼
 VM 1     VM 2
 │         │
OS        OS
 │         │
Kernel    Kernel
 │         │
App       App
```

So:

```text
VM 1 → its own kernel
VM 2 → its own kernel
```

## Container

Containers normally share the host Linux kernel.

```text
Physical Hardware
      │
      ▼
Linux Kernel
      │
 ┌────┼────┐
 ▼    ▼    ▼
 C1   C2   C3
```

Therefore containers are generally lighter than VMs.

---

# 6. But How Does the Container Stay Isolated?

This is where the **Linux Kernel** becomes important.

The kernel provides mechanisms such as:

```text
Namespaces
cgroups
Capabilities
seccomp
```

You don't need to memorize all of them immediately.

For now remember two:

```text
Namespaces → What can the container SEE?

cgroups → How much can the container USE?
```

---

# 7. Namespaces — Simple Example

Suppose your Linux machine has:

```text
PID 1
PID 2
PID 3
PID 4
PID 5
```

A container should not simply see every process on the machine.

Linux namespaces give the container an isolated view.

Container A might see:

```text
PID 1
PID 20
PID 25
```

Container B can also have:

```text
PID 1
PID 20
```

They are not actually the same processes.

They are in different PID namespaces.

Think:

```text
Same Linux Kernel
       │
 ┌─────┴─────┐
 ▼           ▼
Container A  Container B
PID view     PID view
```

---

# 8. cgroups — Simple Example

Suppose you have two containers.

```text
Container A
CPU = 1 core
RAM = 512 MB

Container B
CPU = 2 cores
RAM = 1 GB
```

Linux cgroups can enforce these resource limits.

So remember:

```text
Namespaces → Isolation

cgroups → Resource limits
```

---

# 9. Now the Confusing Part: Docker CLI, Engine, containerd and runc

When you type:

```bash
docker run nginx
```

it looks like the `docker` command itself creates the container.

It doesn't.

There are multiple layers.

The simplified flow is:

```text
You
 │
 ▼
Docker CLI
 │
 ▼
Docker Engine / dockerd
 │
 ▼
containerd
 │
 ▼
runc
 │
 ▼
Linux Kernel
 │
 ▼
Container process
```

Now let's understand each one using a real-world analogy.

---

# 10. Docker CLI

The Docker CLI is what YOU interact with.

Examples:

```bash
docker run nginx
docker ps
docker stop mycontainer
docker images
docker pull nginx
```

The CLI is mainly a **client**.

It sends requests to Docker.

For example:

```text
You
 │
 │ docker run nginx
 ▼
Docker CLI
 │
 │ request
 ▼
Docker Engine
```

The CLI does not itself perform all the low-level container creation.

---

# 11. Docker Daemon — dockerd

`dockerd` is a background process.

Think of it as the **main Docker manager**.

When you say:

```bash
docker run nginx
```

the CLI sends the request to `dockerd`.

Conceptually:

```text
Docker CLI
    │
    │ "Run nginx"
    ▼
 dockerd
    │
    │ "Okay, I'll handle it"
    ▼
containerd
```

`dockerd` manages things such as:

- Containers
- Images
- Networks
- Volumes
- Container lifecycle
- Docker API requests

---

# 12. Docker Engine vs Docker Daemon

This causes confusion because people use the terms interchangeably.

A simple way to remember:

```text
Docker Engine
    │
    └── Docker's server-side/container management functionality
            │
            └── dockerd
```

In interviews, it is generally okay to say:

> Docker Engine is the core Docker component, and `dockerd` is its main daemon process.

You will often hear people casually say:

> "Docker Engine is dockerd."

For your learning, don't get stuck on this distinction.

Focus on the responsibility:

```text
Docker Engine / dockerd
        ↓
High-level Docker management
```

---

# 13. Why Do We Need containerd?

Now suppose `dockerd` has received:

```bash
docker run nginx
```

Docker needs something responsible for managing the container lifecycle.

This is where **containerd** comes in.

Think:

```text
Docker Engine
      │
      ▼
  containerd
```

containerd handles container-related lifecycle work such as:

```text
Create
Start
Stop
Delete
Manage container state
Manage images
Talk to lower-level runtime
```

So:

```text
Docker Engine = Higher-level Docker management

containerd = Container lifecycle management
```

---

# 14. Why Doesn't Docker Engine Directly Talk to runc?

You may ask:

> If runc is the thing that actually creates the container, why not just do:

```text
Docker Engine
      ↓
    runc
```

Technically, you could design a system differently.

But Docker separates responsibilities.

Imagine a company:

```text
Manager
   ↓
Team Lead
   ↓
Worker
```

The manager doesn't need to personally perform every low-level operation.

Similarly:

```text
Docker Engine
      ↓
  containerd
      ↓
     runc
```

Each layer has a focused responsibility.

This also makes the architecture more reusable and maintainable.

---

# 15. runc — The Low-Level Runtime

Now we reach `runc`.

`runc` is much closer to the Linux kernel.

Its job is essentially:

> Create and start the container process using Linux container mechanisms.

Conceptually:

```text
containerd
    │
    ▼
   runc
    │
    ▼
Linux Kernel
```

runc uses Linux mechanisms such as:

```text
Namespaces
cgroups
Capabilities
seccomp
```

So remember:

```text
containerd → manages container lifecycle

runc → actually creates/starts the low-level container process
```

---

# 16. The Complete Docker Command Example

Let's follow:

```bash
docker run nginx
```

Step by step.

## Step 1 — You type the command

```bash
docker run nginx
```

## Step 2 — Docker CLI receives it

```text
Docker CLI
```

The CLI sends a request.

## Step 3 — dockerd receives the request

```text
dockerd
```

It understands:

> User wants an nginx container.

## Step 4 — Docker Engine works with containerd

```text
dockerd
   ↓
containerd
```

containerd manages the container lifecycle operation.

## Step 5 — containerd uses runc

```text
containerd
   ↓
runc
```

## Step 6 — runc asks the Linux kernel to create the isolated process

```text
runc
 ↓
Linux Kernel
```

The kernel provides things like:

```text
Process isolation
Network isolation
Filesystem isolation
CPU limits
Memory limits
```

## Final result

```text
nginx process
running inside a container
```

So the complete mental model is:

```text
docker run nginx
      │
      ▼
Docker CLI
      │
      ▼
Docker Engine / dockerd
      │
      ▼
containerd
      │
      ▼
runc
      │
      ▼
Linux Kernel
      │
      ▼
nginx container process
```

---

# 17. Why Can't CLI Directly Call containerd?

Another common question.

You might think:

```text
Docker CLI
    ↓
containerd
    ↓
runc
```

Why do we need Docker Engine?

Because Docker CLI provides a **Docker user experience and API**, while Docker Engine handles higher-level Docker concepts.

For example:

```text
docker run
docker build
docker pull
docker network
docker volume
docker compose
```

Docker Engine coordinates these Docker-level operations.

containerd is focused more specifically on container runtime/lifecycle responsibilities.

So:

```text
CLI
 ↓
Docker-level API/management
 ↓
containerd
 ↓
low-level runtime
```

---

# 18. Image Layers

Docker images are generally built using layers.

For example:

```text
Java Application Image
│
├── Application layer
├── Dependency layer
├── Java layer
└── Base filesystem layers
```

Suppose you create:

```text
Container A
Container B
Container C
```

from the same image.

The read-only image layers can be shared.

Each container gets its own writable layer.

Conceptually:

```text
             Image Layers
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Container A Container B Container C
   writable     writable     writable
   layer        layer        layer
```

This is why Docker does not need three completely independent copies of the same image.

---

# 19. Important Question: Is Java Installed Three Times?

Suppose your image contains Java.

```text
Java Image
│
├── Java
├── Application
└── Dependencies
```

You run three containers:

```text
C1
C2
C3
```

They use the same underlying image layers.

So conceptually:

```text
             Java Image
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
      C1         C2         C3
```

You should think:

> The containers can share the image's read-only layers.

They are still separate running environments/processes.

---

# 20. Docker Networking

Containers can communicate with each other through Docker networking.

Example:

```text
Java Application Container
          │
          │ Docker Network
          │
          ▼
     Redis Container
```

Another example:

```text
Java App
   │
   ├── MySQL
   │
   └── Redis
```

Docker provides network drivers such as:

```text
bridge
host
none
overlay
```

For basic local Docker usage, `bridge` is commonly encountered.

---

# 21. Docker Volumes

Containers are usually replaceable.

Suppose you run MySQL:

```text
MySQL Container
```

and MySQL stores data inside the container filesystem.

If the container is deleted, that data may disappear.

So we use a volume:

```text
MySQL Container
       │
       ▼
Docker Volume
       │
       ▼
Persistent Data
```

Now:

```text
Delete container ❌

Volume          ✅
Data            ✅
```

This is why databases commonly use persistent storage.

---

# 22. Docker Desktop

Now let's move to your Windows machine.

Docker itself needs a Linux environment to run Linux containers.

Docker Desktop makes this much easier on Windows.

A simplified mental model is:

```text
Windows
   │
   ▼
Docker Desktop
   │
   ▼
WSL2
   │
   ▼
Linux Environment / Kernel
   │
   ▼
Docker Components
   │
   ▼
Linux Containers
```

Docker Desktop provides convenient integration for:

- Docker CLI
- Docker Engine
- Images
- Containers
- Networking
- Volumes
- GUI
- WSL2 integration

---

# 23. What Is WSL2?

WSL means:

> Windows Subsystem for Linux

WSL2 provides a Linux environment inside Windows using virtualization technology and a real Linux kernel.

For Docker's Linux-container scenario, the important idea is:

```text
Windows
   │
   ▼
WSL2
   │
   ▼
Linux Kernel
   │
   ▼
Linux Containers
```

So Docker Desktop and WSL2 are not the same thing.

Think:

```text
Docker Desktop
= Docker platform/tooling/integration

WSL2
= Linux environment/kernel mechanism on Windows
```

---

# 24. Does Docker Desktop Create One Linux OS Per Container?

No.

Suppose you run:

```text
Java
Kafka
Redis
```

You should visualize:

```text
             Linux Kernel
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
      Java       Kafka      Redis
   Container   Container   Container
```

Not:

```text
Java  → Linux OS
Kafka → Linux OS
Redis → Linux OS
```

There is a shared underlying Linux kernel.

---

# 25. Now What Is Kubernetes?

Docker is very useful when running containers.

But imagine your company has:

```text
100 microservices
500 containers
20 servers
```

Now someone has to answer:

```text
Where should each container run?

What if a container crashes?

How many copies should run?

How do we deploy a new version?

How do services find each other?

How do we scale from 5 containers to 20?

How do we replace failed containers?
```

Doing this manually is difficult.

This is where an **orchestrator** comes in.

---

# 26. Orchestrator — Simple Meaning

An orchestrator is like a **manager for containers across multiple machines**.

The most common example is Kubernetes.

Think:

```text
Docker
= Run containers

Kubernetes
= Manage many containers across machines
```

---

# 27. Kubernetes Desired State

Suppose you tell Kubernetes:

```text
I want 5 instances of order-service.
```

Kubernetes stores that as the desired state:

```text
Desired = 5
```

Suppose one container crashes:

```text
Desired = 5
Actual = 4
```

Kubernetes notices the difference.

It creates another workload:

```text
4 → 5
```

So the goal is:

```text
Desired State = Actual State
```

This is one of the most important Kubernetes concepts.

---

# 28. Kubernetes Cluster

A simplified cluster looks like:

```text
             Kubernetes
             Control Plane
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Worker 1          Worker 2
```

The control plane makes decisions such as:

```text
Where should workloads run?

How many replicas are required?

What is the desired state?
```

Worker nodes actually run the workloads.

---

# 29. Worker Node

A worker node can be visualized as:

```text
Worker Node
│
├── kubelet
│
├── container runtime
│      │
│      └── containerd
│             │
│             └── runc
│
└── Pods / Containers
```

---

# 30. What Is kubelet?

`kubelet` is an agent running on each worker node.

Its simple responsibility is:

> Make sure the workloads assigned to this node are running correctly.

Think:

```text
Kubernetes Control Plane
          │
          │ "Run this workload here"
          ▼
       kubelet
          │
          ▼
   Container Runtime
          │
          ▼
     Containers
```

The kubelet is the node-level agent that makes sure the desired workloads assigned to its node are running.

---

# 31. Does Kubernetes Need Docker Engine?

Modern Kubernetes does **not** require Docker Engine.

This is important.

A common modern architecture is:

```text
Kubernetes
     │
     ▼
  kubelet
     │
     ▼
 containerd
     │
     ▼
    runc
     │
     ▼
Linux Kernel
```

So Kubernetes can work directly with a supported container runtime such as containerd.

You should remember:

```text
Docker Engine ≠ Required Kubernetes component
```

Docker Engine can use containerd internally, but Kubernetes can use a container runtime directly.

---

# 32. Docker vs Kubernetes

| Docker | Kubernetes |
|---|---|
| Builds images | Uses container images |
| Runs containers | Manages workloads |
| Mainly local/container platform | Cluster/container orchestration |
| `docker` CLI | `kubectl` |
| Docker Engine | Kubernetes control plane + node components |
| Container lifecycle | Scheduling, scaling, healing, deployment |

Easy memory:

```text
Docker = Build & Run Containers

Kubernetes = Manage Containers at Scale
```

---

# 33. The Whole Architecture — Docker Only

For a normal Docker environment:

```text
              YOU
               │
               │ docker run
               ▼
          Docker CLI
               │
               ▼
       Docker Engine / dockerd
               │
               ▼
           containerd
               │
               ▼
              runc
               │
               ▼
          Linux Kernel
               │
               ▼
        Container Process
```

---

# 34. The Whole Architecture — Windows + Docker Desktop

For your Windows laptop, keep this mental model:

```text
Windows
   │
   ▼
Docker Desktop
   │
   ▼
WSL2
   │
   ▼
Linux Kernel
   │
   ▼
Docker Engine / dockerd
   │
   ▼
containerd
   │
   ▼
runc
   │
   ▼
Linux Containers
   │
   ├── Java
   ├── Kafka
   └── Redis
```

> The exact internal implementation can vary by Docker Desktop version/configuration. This is the conceptual model you need for understanding the architecture.

---

# 35. The Whole Architecture — Kubernetes

In production, think:

```text
                 Kubernetes
                 Control Plane
                      │
              "I want 5 replicas"
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Worker 1    Worker 2    Worker 3
          │           │           │
       kubelet     kubelet     kubelet
          │           │           │
      containerd  containerd  containerd
          │           │           │
        runc        runc        runc
          │           │           │
      Linux Kernel Linux Kernel Linux Kernel
          │           │           │
      Containers   Containers   Containers
```

Each worker machine has its own Linux kernel.

---

# 36. Very Important: Kubernetes vs Docker Architecture

Do not mix these two pictures.

## Local Docker

```text
You
 ↓
Docker CLI
 ↓
Docker Engine
 ↓
containerd
 ↓
runc
 ↓
Linux Kernel
 ↓
Container
```

## Kubernetes

```text
Kubernetes Control Plane
 ↓
Worker Node
 ↓
kubelet
 ↓
containerd
 ↓
runc
 ↓
Linux Kernel
 ↓
Container
```

The major difference is:

```text
Docker Engine
→ provides Docker's high-level container management

Kubernetes
→ decides where workloads should run and maintains desired state
```

---

# 37. Who Does What?

This table is worth remembering.

| Component | Simple Responsibility |
|---|---|
| Docker CLI | Takes commands from you |
| Docker Engine / dockerd | High-level Docker management |
| containerd | Manages container lifecycle |
| runc | Creates/starts the low-level container process |
| Linux Kernel | Provides processes, isolation, CPU, memory, networking, etc. |
| Docker Desktop | Makes Docker convenient on Windows/macOS |
| WSL2 | Provides Linux environment/kernel mechanism on Windows |
| Kubernetes | Orchestrates workloads across machines |
| kubelet | Ensures workloads assigned to a node are running |

---

# 38. Interview Questions You Should Be Able to Answer

## Q1. What happens when you run `docker run nginx`?

Answer:

```text
docker CLI
   ↓
Docker Engine / dockerd
   ↓
containerd
   ↓
runc
   ↓
Linux Kernel
   ↓
nginx process
```

In simple words:

> The CLI sends the request to Docker Engine. Docker Engine works with containerd, containerd uses the low-level runtime such as runc, and runc creates the container process using Linux kernel mechanisms.

---

## Q2. Why do we need containerd?

Answer:

> containerd handles container lifecycle and runtime management between the higher-level Docker Engine and the low-level runtime.

Simple picture:

```text
Docker Engine
      ↓
containerd
      ↓
runc
```

---

## Q3. Why do we need runc?

Answer:

> runc is the low-level OCI runtime that creates and starts the container process using Linux kernel isolation mechanisms.

Simple picture:

```text
containerd
    ↓
  runc
    ↓
Linux Kernel
```

---

## Q4. Does every container have its own OS?

Answer:

> No. Containers normally share the underlying Linux kernel. They have their own isolated user-space filesystem and processes.

---

## Q5. What is the difference between namespace and cgroup?

Easy answer:

```text
Namespace → What can the container see?

cgroup → How much resource can the container use?
```

---

## Q6. What is Docker Desktop?

Answer:

> Docker Desktop is a desktop platform that provides Docker tooling and integration on Windows and macOS. On Windows, it can use WSL2 to provide the Linux environment needed for Linux containers.

---

## Q7. What is WSL2?

Answer:

> WSL2 stands for Windows Subsystem for Linux 2. It provides a Linux environment and Linux kernel mechanism inside Windows, which can be used to run Linux containers.

---

## Q8. Does Kubernetes need Docker Engine?

Answer:

> Modern Kubernetes does not require Docker Engine. It can use a supported container runtime such as containerd directly through the kubelet.

---

## Q9. Who decides where a container should run in Kubernetes?

Answer:

> The Kubernetes control plane, particularly the scheduler, decides where workloads should run.

---

## Q10. Who makes sure the workload is running on a node?

Answer:

> kubelet.

---

# 39. The Simplest Mental Model

If all the above still feels complicated, remember only this first:

```text
IMAGE
  │
  │ creates
  ▼
CONTAINER
  │
  │ managed by
  ▼
CONTAINER RUNTIME
  │
  ▼
Linux Kernel
  │
  ▼
Hardware
```

For Docker:

```text
Docker CLI
    ↓
Docker Engine
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
    ↓
Container
```

For Kubernetes:

```text
Kubernetes
    ↓
kubelet
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
    ↓
Container
```

For your Windows laptop:

```text
Windows
   ↓
Docker Desktop
   ↓
WSL2
   ↓
Linux Kernel
   ↓
Docker components
   ↓
Containers
```

---

# 40. One Final Analogy

Imagine a restaurant.

```text
You
 ↓
Waiter
 ↓
Manager
 ↓
Kitchen Manager
 ↓
Chef
 ↓
Kitchen equipment
```

Map it to Docker:

```text
You
 ↓
Docker CLI
 ↓
Docker Engine / dockerd
 ↓
containerd
 ↓
runc
 ↓
Linux Kernel
```

Each layer has a different job.

```text
Docker CLI
→ "I want nginx"

Docker Engine
→ "Okay, I'll manage this Docker request"

containerd
→ "I'll manage the container lifecycle"

runc
→ "I'll create/start the low-level container process"

Linux Kernel
→ "I'll provide the actual OS-level mechanisms"
```

That is the architecture.

---

# 41. What You Should Memorize for Interviews

Do NOT try to memorize every internal detail.

Remember these 8 lines:

```text
1. Image = read-only template
2. Container = running isolated process created from image
3. Containers normally share the Linux kernel
4. Docker CLI = client
5. Docker Engine/dockerd = high-level Docker management
6. containerd = container lifecycle/runtime management
7. runc = low-level container creation
8. Kubernetes = orchestrates workloads across machines
```

And this diagram:

```text
Docker:

Docker CLI
   ↓
Docker Engine
   ↓
containerd
   ↓
runc
   ↓
Linux Kernel
   ↓
Container


Kubernetes:

Kubernetes Control Plane
   ↓
Worker Node
   ↓
kubelet
   ↓
containerd
   ↓
runc
   ↓
Linux Kernel
   ↓
Container
```

If these two diagrams are clear, the rest of Docker architecture becomes much easier to understand.
