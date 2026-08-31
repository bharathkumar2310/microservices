DOCKER :


1. What is Docker?

        Docker is a platform for packaging and running applications in isolated environments called containers.

The simplest mental model:

    Docker packages your application + everything it needs to run, and allows you to run that package consistently on different machines.

For example, suppose you have a Spring Boot application.

Your application may need:

    Spring Boot application
    +
    Java 21
    +
    specific libraries
    +
    environment variables
    +
    configuration
    +
    OS-level dependencies
    +
    MySQL/Kafka/Redis connections


Without Docker, you install these things manually on your machine.

With Docker, you can package the application into an image and run it as a container.

             Docker Image
        ┌─────────────────────┐
        │ Spring Boot App     │
        │ Java/runtime        │
        │ dependencies        │
        │ configuration       │
        └─────────────────────┘
                  ↓
              Container
                  ↓
             Application

2. What problem existed before Docker?

Imagine your company has:

    Developer laptop
    ↓
    Spring Boot application
    
    The developer says:
    
    "It works on my machine."
    
    But when the application goes to another machine:
    
    Developer machine
    ↓
    Java 21
    ↓
    Application works


    Production machine
    ↓
    Java 17
    ↓
    Application fails
    
    This is the famous:
    
    "Works on my machine" problem.

3. Example: Java version problem

        Developer has:
        
        Java 21
        Spring Boot application
        
        Production has:
        
        Java 17

The application might fail because it was compiled/run expecting Java 21.

You could manually install Java 21 on production.

But then you have another problem:

    Server 1 → Java 21
    Server 2 → Java 21
    Server 3 → Java 21
    Server 4 → Java 21
...

And you need to maintain all of them.

    Docker helps standardize the runtime environment.

4. Another problem: dependency conflicts

Suppose you have two applications.

    Application A
    Java 21
    MySQL client version X
    Library A version 2
    Application B
    Java 17
    MySQL client version Y
    Library A version 1

If both run directly on the same machine, their dependencies can conflict.

Docker lets you isolate them:

    Machine
    │
    ├── Container A
    │     ├── Application A
    │     └── its dependencies
    │
    └── Container B
    ├── Application B
    └── its dependencies

They can coexist without directly interfering with each other.

5. Another huge problem: setting up environments

Imagine a new developer joins your team.

Without Docker:

    Install Java
    Install Maven
    Install MySQL
    Install Redis
    Install Kafka
    Configure MySQL
    Configure Redis
    Configure Kafka
    Configure environment variables
    Configure ports
...

It could take hours or days.

    With Docker, you can provide the required images/configuration and start the environment much more consistently.

For example:

    docker run mysql
    docker run redis
    docker run kafka

Now you have the services running as containers.


------------------------------------------------------------------------------------------------------------------------------------------

DOCKER CONTAINER :

    A Docker container is an isolated environment in which a process/application runs, created and managed by Docker.

The easiest way to understand it is by comparing it with a normal process and a VM.

1. Without Docker

Suppose you run a Spring Boot application:

    Windows/Linux
    │
    └── Java process
    │
    └── Spring Boot application

That Java process uses the host OS directly.

2. With Docker

        Now you package the application into a Docker image and run it:

        Host OS / Linux Kernel
        │
        Docker
        │
        ↓
        ┌─────────────────────┐
        │   Container         │
        │                     │
        │ Spring Boot app     │
        │ Java/runtime        │
        │ filesystem          │
        │ environment         │
        └─────────────────────┘

        The container gives the application an isolated view of things like its filesystem, processes, network, and resources.

        But the container doesn't contain its own Linux kernel.

3. Very important: container ≠ VM

A VM looks roughly like:

        Physical Hardware
        ↓
        Host OS
        ↓
        Hypervisor
        ↓
        ┌─────────────────┐
        │ VM              │
        │ Linux OS        │
        │ Linux Kernel    │
        │ Application     │
        └─────────────────┘

A container looks more like:

        Physical Hardware
        ↓
        Host OS / Linux kernel
        ↓
        Docker
        ↓
        ┌──────────────┐  ┌──────────────┐
        │ Container A  │  │ Container B  │
        │ App          │  │ App          │
        └──────────────┘  └──────────────┘
        │                 │
        └──── shared ─────┘
        kernel

That's why containers are generally much lighter than VMs.

4. So what is actually inside a container?

This is a common misconception.

A container can have its own:

    application
    libraries
    filesystem view
    environment variables
    network interface
    process namespace
    resource limits
    
    But it doesn't have its own kernel.

For example:

    Container
    │
    ├── Spring Boot JAR
    ├── Java runtime
    ├── libraries
    ├── /app
    ├── environment variables
    └── isolated processes
    ↓
    Linux kernel
    ↓
    CPU/RAM

----------------------------------------------------------------------------------------------------------------------------------------

DOCKER IMAGE :


    If container = running application, then Docker image = the packaged blueprint/template used to create that container.

1. Simple relationship

Think:

    Docker Image
    │
    │ docker run
    ↓
    Docker Container

For example:

    springboot-app:1.0
    ↓
    Container 1

You can use the same image to create multiple containers:

             springboot-app:1.0
                 /    |    \
                ↓     ↓     ↓
          Container  Container  Container
              1          2          3
2. What does an image contain?

Suppose you have a Spring Boot application.

You need:

    Spring Boot JAR
    Java runtime
    Application libraries
    Configuration
    Required files

A Docker image packages the application's filesystem contents and metadata needed to create the container.

Conceptually:

    Docker Image
    │
    ├── Application JAR
    ├── Java runtime
    ├── Libraries
    ├── Application files
    ├── Environment/config metadata
    └── Startup command

Then Docker uses this image to create the container.

3. Image is NOT a running thing

This distinction is very important.

    IMAGE
    ↓
    static/package/template
    ↓
    CONTAINER
    ↓
    running process

For example:

    nginx image
    ↓
    container 1 → running nginx
    container 2 → running nginx
    container 3 → running nginx

The image itself isn't "running."

4. Where does the image come from?

Usually you create it using a Dockerfile.

Example:

    FROM eclipse-temurin:21
    COPY app.jar app.jar
    ENTRYPOINT ["java", "-jar", "app.jar"]

Then:

    Dockerfile
    ↓
    docker build
    ↓
    Docker Image
    ↓
    docker run
    ↓
    Container

So:

    Dockerfile = instructions
    Image      = packaged result
    Container  = running instance

5. What is FROM doing?

Suppose:

    FROM eclipse-temurin:21

You're saying:

    Start my image using an existing Java 21 image as the base.

Then you add your application:

    Java 21 base image
    +
    your Spring Boot JAR
    +
    configuration
    ↓
    your Docker image

6. Image vs container

   | Docker Image               | Docker Container               |
   | -------------------------- | ------------------------------ |
   | Package/template           | Running instance               |
   | Doesn't run by itself      | Runs a process                 |
   | Built from Dockerfile      | Created from image             |
   | Can be reused              | Has its own runtime state      |
   | Immutable/read-only layers | Has a writable container layer |


For example:

             Image
       order-service:1.0
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
    C1       C2      C3
    running  running  running

-------------------------------------------------------------------------------------------------------------------------------

DOCKER DESKTOP



## 1. What is Docker Desktop?

      Docker Desktop is a developer-friendly application that makes it easy to run and manage containers on Windows and macOS.

      It provides/configures several components needed to work with Docker, such as:
      
      - Docker Engine
        - Docker CLI
        - Container runtime
        - Networking and storage integration
        - Integration with WSL2 on Windows
      
      > Docker Desktop itself is NOT an operating system and does NOT create a kernel.

---

# 2. What is WSL?

      WSL = **Windows Subsystem for Linux**.

      It is a Windows feature that allows you to run a Linux environment inside Windows.

      You can run Linux applications and commands without replacing Windows with Linux.

For example:

```text
Windows
│
├── Windows applications
│
└── WSL2
     │
     ├── Linux user space
     └── Linux kernel