# Server and Virtual Machine — Notes

## 1. What is a Server?

A **server is a computer** whose job is to provide resources or services to other computers/applications.

A server can have:
- CPU
- RAM
- Storage
- Network interface
- Operating system
- Applications/services

Example:

```text
Client
  |
  v
Server
  |
  +-- CPU
  +-- RAM
  +-- Storage
  +-- Network
  +-- OS
  +-- Applications
```

A normal laptop is also a computer. The difference is mainly **how it is used**. A powerful computer used to serve applications to many clients is commonly called a server.

---

## 2. Why do we use Data Centers?

You can deploy an application on your local PC.

For example:

```text
Your Laptop
   |
   +-- Spring Boot application
   +-- Database
```

But production systems need much more:

- High CPU/RAM capacity
- Large and reliable storage
- High-speed networking
- Redundancy
- Backup power
- Cooling
- Monitoring
- Physical security
- High availability
- Ability to handle many users
- Ability to scale

A **data center is essentially a large facility containing many servers and supporting infrastructure**.

Think:

```text
Data Center
 |
 +-- Server 1
 +-- Server 2
 +-- Server 3
 +-- ...
 +-- Networking
 +-- Storage
 +-- Power
 +-- Cooling
 +-- Backup systems
```

So yes, at a basic level, a server is still a computer — a data center simply provides computers and infrastructure at large scale and with reliability.

---

# 3. Can Multiple Applications Run on One Server?

Yes.

A single physical server can run multiple applications:

```text
Physical Server
 |
 +-- User Service
 +-- Order Service
 +-- Payment Service
```

However, applications can compete for:
- CPU
- RAM
- Disk
- Network
- Ports

They can also have conflicting dependencies.

Example:

```text
Application A → Java 17
Application B → Java 21
```

Or:

```text
Application A → library version 1
Application B → library version 2
```

There can also be failures or resource contention.

This led to the need for better isolation and resource management.

---

# 4. What is a Virtual Machine (VM)?

A **Virtual Machine is a software-created computer**.

Instead of buying another physical computer, virtualization software creates a virtual computer using the resources of an existing physical machine.

Example:

```text
Physical Server
      |
      v
  Hypervisor
      |
  +---+---+---+
  |   |   |   |
 VM1 VM2 VM3
```

Each VM behaves like an independent computer.

---

# 5. What is a Hypervisor?

A **hypervisor** is software (or firmware/software layer) that creates and manages VMs.

It allocates physical resources to VMs.

Example:

```text
Physical Server
 |
 |-- 16 CPU cores
 |-- 64 GB RAM
 |-- 1 TB SSD
 |
 v
Hypervisor
 |
 +-- VM 1 → 4 vCPU, 16 GB RAM
 +-- VM 2 → 4 vCPU, 16 GB RAM
 +-- VM 3 → 4 vCPU, 16 GB RAM
```

The resources are ultimately backed by the same physical hardware.

---

# 6. What does "Virtual" mean?

**Virtual means software-created representation of something that behaves like a physical resource.**

Examples:

```text
Virtual CPU    → uses/schedules work on physical CPU
Virtual RAM    → represents memory allocated from physical RAM
Virtual Disk   → can be backed by a file on physical storage
Virtual NIC    → software representation of a network interface
```

There isn't necessarily separate physical hardware for each VM.

For example:

```text
Physical CPU
     |
 Hypervisor
     |
 +---+---+
 |       |
VM1     VM2
4 vCPU  4 vCPU
```

The hypervisor schedules the virtual CPUs onto available physical CPU resources.

A VM's virtual CPU is **not necessarily a permanently reserved physical CPU core**.

---

# 7. Does a VM Have Its Own Memory?

A VM can be configured with its own **virtual memory allocation**.

Example:

```text
Physical Server
32 GB RAM
     |
 Hypervisor
     |
 +---+----------------+
 |                    |
VM1                  VM2
8 GB                 8 GB
```

Both ultimately use physical RAM.

The VM's memory is isolated from other VMs through virtualization mechanisms.

So:

> Hardware resources are shared underneath, but the VM gets an isolated virtual view of those resources.

---

# 8. Why Does a VM Need an OS?

A VM is designed to behave like a **complete computer**.

Therefore it needs an operating system to manage its virtual resources and run applications.

Example:

```text
Physical Hardware
       |
   Hypervisor
       |
   Virtual Hardware
       |
   Guest OS
       |
   Application
```

The guest OS manages things such as:
- Processes
- Memory
- Files
- Networking
- CPU scheduling
- Devices

---

# 9. What is an Operating System?

An **Operating System (OS)** is system software that manages computer resources and provides services to applications.

Examples:
- Windows
- Linux distributions such as Ubuntu
- macOS

Conceptually:

```text
Applications
     |
     v
Operating System
     |
     v
Hardware
```

The OS manages resources such as:
- CPU
- RAM
- Storage
- Networking
- Devices
- Processes

---

# 10. What is the Kernel?

The **kernel is the core part of an operating system**.

It is responsible for low-level management of resources such as:
- CPU
- RAM
- Processes
- Storage
- Networking
- Hardware devices

A simplified model:

```text
Applications
     |
     v
OS components / system interfaces
     |
     v
Kernel
     |
     v
Hardware
```

Important:

> The kernel is not a completely separate thing from the OS. The kernel is the core of the OS.

For example:

```text
Ubuntu/Linux-based OS
 |
 +-- Linux kernel
 +-- Libraries
 +-- Utilities
 +-- Services
 +-- Other user-space software
```

---

# 11. What is a Process?

A **process is a running instance of a program**.

For example:

```text
Program on disk
      |
   Run it
      |
      v
Running process
```

If you start a Spring Boot application, the running application becomes a process.

The kernel manages processes and schedules them on CPU resources.

---

# 12. How Does a VM Get a CPU?

Suppose the physical server has:

```text
8 physical CPU cores
```

A VM is configured with:

```text
4 virtual CPUs
```

The hypervisor schedules the VM's virtual CPU work onto physical CPU resources.

For example, at one moment:

```text
vCPU 1 → physical core 2
vCPU 2 → physical core 4
vCPU 3 → physical core 6
vCPU 4 → physical core 7
```

Later, the work may run on different cores.

CPU pinning/affinity can be configured in advanced scenarios, but normally virtual CPUs are scheduled dynamically.

---

# 13. If Hardware Is Shared, Why Do We Say VMs Are Isolated?

**Isolation does not mean separate physical hardware.**

It means one VM should not freely access or interfere with another VM's resources/environment.

Example:

```text
Same physical RAM
       |
   Hypervisor
    /         v        v
 VM1       VM2
```

VM1 should not simply read or modify VM2's memory.

Similarly, VMs have isolated:
- Processes
- Memory
- Virtual disks
- Network environments
- Users/permissions

So remember:

> **Physical resources can be shared; the virtual environments are isolated.**

---

# 14. What Problem Did VMs Solve?

Before virtualization, you might have:

```text
Server 1 → Application A
Server 2 → Application B
Server 3 → Application C
```

This could waste resources because one server might be underutilized.

Virtualization allows:

```text
One Physical Server
       |
   Hypervisor
       |
 +-----+-----+-----+
 |           |     |
VM1         VM2   VM3
App A       App B App C
```

Benefits:
- Better hardware utilization
- Isolation
- Independent OS environments
- Ability to run different operating systems
- Easier provisioning and management

---

# 15. VM Does NOT Mean Completely Separate Physical Hardware

This is important.

A VM:

```text
VM
 |
 +-- Virtual CPU
 +-- Virtual RAM
 +-- Virtual Disk
 +-- Virtual Network
```

These are backed by the physical machine.

For example:

```text
Virtual Disk
     |
 Hypervisor
     |
Physical SSD
```

A virtual disk can be represented by a file or other storage allocated on physical storage.

---

# 16. Example: Windows Running Linux VM

Your laptop can have:

```text
Physical Hardware
       |
     Windows
       |
 VirtualBox / VMware / Hyper-V
       |
   Virtual Machine
       |
     Ubuntu
       |
  Applications
```

Windows is the host OS in this example.

Ubuntu is the **guest OS**.

The VM gets virtual hardware from the virtualization layer.

---

# 17. Why Does a VM Need Its Own OS Instead of Sharing the Host OS?

Because the VM is intended to be a **complete independent computer environment**.

Example:

```text
Host
Windows
 |
 +-- VM1 → Ubuntu
 |
 +-- VM2 → Ubuntu
 |
 +-- VM3 → Windows Server
```

Each guest OS has its own:
- Kernel
- Processes
- Filesystem
- Users/permissions
- Networking environment
- System libraries

This allows different environments to coexist.

For example:

```text
VM1 → Ubuntu + Java 17
VM2 → Ubuntu + Java 21
VM3 → Windows Server + Windows application
```

---

# 18. What is the Kernel's Relationship With Applications?

Applications don't normally directly control hardware.

For example:

```text
Java Application
      |
      v
System/library interfaces
      |
      v
Kernel
      |
      v
CPU / RAM / Disk / Network
```

If an application needs memory, it requests memory through the OS/kernel mechanisms.

If it needs to write a file, it requests filesystem operations.

If it needs CPU time, the kernel schedules its process.

---

# 19. System Calls

A **system call** is a controlled way for a user application to request a service from the kernel.

Conceptually:

```text
Application
    |
    | "I need to open this file"
    v
System Call
    |
    v
Kernel
    |
    v
Storage
```

Or:

```text
Application
    |
    | "I need memory"
    v
System Call
    |
    v
Kernel
    |
    v
RAM
```

You do not need to memorize specific system calls yet.

---

# 20. Where Docker Enters

Now we can understand the motivation for containers.

A VM looks like:

```text
Physical Hardware
       |
   Hypervisor
       |
   VM
       |
 Guest OS + Kernel
       |
 Application
```

If you have many applications/microservices, having a complete guest OS for every application can introduce overhead.

Containers take a different approach:

```text
Physical Hardware
       |
   Host OS / Kernel
       |
 Container Runtime
       |
 +-----+-----+-----+
 |           |     |
 C1          C2    C3
App A       App B App C
```

The containers share the underlying kernel instead of each requiring a separate guest kernel.

This is why containers are generally lighter than VMs.

---

# 21. Your Windows + Docker Desktop Situation

Your main OS is still **Windows**.

For Linux containers, Docker Desktop provides a Linux environment underneath Windows, commonly using WSL2.

Simplified:

```text
Physical Laptop
       |
    Windows
       |
 Docker Desktop
       |
 Linux environment / VM
       |
 Linux kernel
       |
 +-----+-----+-----+
 |           |     |
Container  Container Container
```

You do NOT need to replace Windows with Linux.

---

# 22. Does Every Container Have Its Own OS?

No — not a complete guest OS/kernel.

A container can contain its own:
- Application
- Libraries
- Dependencies
- Utilities
- Filesystem/user-space files

But Linux containers share the underlying Linux kernel.

Example:

```text
             SAME LINUX KERNEL
                    |
       +------------+------------+
       |            |            |
       v            v            v
   Container 1  Container 2  Container 3
     User         Payment       Order
```

A container may use an Ubuntu-based or Alpine-based image, but that does not mean it contains a separate Linux kernel.

---

# 23. How Does a Container Communicate With the Kernel?

There isn't a special Docker system-call router.

The application inside the container still uses the normal system-call interface:

```text
Application
     |
Libraries / system interface
     |
System call
     |
Shared Linux kernel
     |
Hardware
```

Docker creates/configures the isolated environment.

The Linux kernel performs the actual resource management and isolation.

Linux kernel mechanisms include:
- Namespaces → isolation of processes, networking, filesystem views, etc.
- cgroups → resource control such as CPU and memory

You don't need to master these yet.

---

# 24. Why Linux on a Windows Laptop?

Your laptop's main OS is Windows:

```text
Windows
└── Windows kernel
```

But Linux containers require a Linux kernel.

Therefore Docker Desktop provides a Linux environment underneath Windows:

```text
Windows
   |
Docker Desktop
   |
WSL2 / Linux VM
   |
Linux kernel
   |
Linux containers
```

Your Windows installation remains your main OS.

---

# 25. The Key Mental Model

### Physical computer

```text
Hardware
   |
OS
   |
Kernel
   |
Applications / Processes
```

### VM

```text
Physical Hardware
   |
Hypervisor
   |
Virtual Hardware
   |
Guest OS
   |
Guest Kernel
   |
Application
```

### Containers

```text
Physical Hardware
   |
Host / underlying OS
   |
Host / underlying Kernel
   |
Container Runtime
   |
Containers
   |
Applications
```

### The most important distinction

> **VM = virtualizes a complete computer, including a guest OS/kernel.**

> **Container = isolates an application environment while sharing the underlying compatible kernel.**

---

# 26. Final Picture

The evolution we have learned so far:

```text
                 PHYSICAL SERVER
                       |
                       v
              ┌─────────────────┐
              │ Hardware        │
              │ CPU / RAM / SSD │
              │ Network         │
              └────────┬────────┘
                       |
                       v
                  Host OS
                       |
                       v
                   Hypervisor
                       |
          ┌────────────┼────────────┐
          v            v            v
        VM 1          VM 2         VM 3
          |            |            |
      Guest OS      Guest OS     Guest OS
          |            |            |
        App A        App B       App C
```

VMs gave us **isolation and better hardware utilization**, but every VM needs a guest OS/kernel.

Containers later allow:

```text
                 PHYSICAL SERVER
                       |
                       v
                    Host OS
                       |
                       v
                 Shared Kernel
                       |
                       v
                Container Runtime
                       |
          ┌────────────┼────────────┐
          v            v            v
      Container    Container    Container
        App A        App B        App C
```

So the big idea is:

> **VMs isolate complete machines. Containers isolate application environments while sharing the kernel.**

This is the foundation we need before learning **Docker images, Dockerfiles, containers, volumes, networks, and Docker commands**.
