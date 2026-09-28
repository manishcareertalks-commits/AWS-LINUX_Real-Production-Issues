# Docker Architecture --- Complete Study Notes

<img width="979" height="858" alt="image" src="https://github.com/user-attachments/assets/a81a2091-b29a-401f-a5fb-e5df0f7fd628" />


## 1. What is Docker Architecture?

Docker uses a **client-server architecture** to build, distribute, and
run containers.

The main flow is:

``` text
Developer
    ↓
Docker CLI
    ↓
Docker API
    ↓
Docker Daemon (dockerd)
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
    ↓
Docker Container
    ↓
Application
```

Docker separates responsibilities across different components:

-   **Docker CLI** sends commands.
-   **Docker API** provides communication between the client and daemon.
-   **Docker Daemon (`dockerd`)** manages Docker resources and
    operations.
-   **Docker Registry** stores and distributes images.
-   **containerd** manages the container lifecycle.
-   **runc** creates and starts the container process.
-   **Linux Kernel** provides the underlying isolation and
    resource-control mechanisms.
-   **Container** provides the runtime environment for the application.

> **Core interview concept:**
> `dockerd → containerd → runc → Linux Kernel`

------------------------------------------------------------------------

# 2. Docker Architecture Overview

Docker architecture can be understood as a chain of responsibilities.

``` text
                         Developer
                             |
                             v
                        Docker CLI
                             |
                        Docker API
                             |
                             v
                    Docker Daemon
                       (dockerd)
                             |
              +--------------+--------------+
              |                             |
              v                             v
       Docker Registry                Local Image
       (if required)                       |
              |                             |
              +--------------+--------------+
                             |
                             v
                         containerd
                             |
                             v
                           runc
                             |
                             v
                       Linux Kernel
                             |
                             v
                        Container
                             |
                             v
                        Application
```

### Important clarification

The **Docker Registry is not always in the execution path**.

If the required image already exists locally, Docker can use the local
image directly.

``` text
Image available locally
        ↓
    Use image

Image unavailable
        ↓
Pull from Registry
        ↓
    Use image
```

------------------------------------------------------------------------

# 3. Major Docker Architecture Components

## 3.1 Docker Client / Docker CLI

The **Docker CLI** is the interface used by developers and
administrators to interact with Docker.

Commands such as:

``` bash
docker run nginx
docker ps
docker stop <container>
```

are entered through the CLI.

The CLI sends these requests to the Docker Daemon through the Docker
API.

### Remember

> **Docker CLI = Sends Docker commands**

``` text
Developer
    ↓
Docker CLI
    ↓
Docker API
    ↓
dockerd
```

------------------------------------------------------------------------

## 3.2 Docker API

The **Docker API** is the communication interface between the Docker
Client and Docker Daemon.

When a command is executed through the Docker CLI, the CLI communicates
with `dockerd` through the Docker API.

``` text
Docker CLI
     ↓
Docker API
     ↓
Docker Daemon
```

### Remember

> **Docker API = Communication layer between the client and daemon**

------------------------------------------------------------------------

## 3.3 Docker Daemon --- `dockerd`

The **Docker Daemon** is the main Docker management component.

It receives requests from the Docker Client and manages Docker resources
and operations such as:

-   Images
-   Containers
-   Networks
-   Volumes
-   Container creation and execution

For container execution, `dockerd` works with **containerd**, which in
turn works with **runc**.

``` text
Docker CLI
     ↓
   dockerd
     ↓
 containerd
     ↓
    runc
```

### Remember

> **dockerd = Main Docker management layer**

------------------------------------------------------------------------

## 3.4 Docker Registry

A **Docker Registry** is a storage and distribution system for Docker
images.

Examples include:

-   Docker Hub
-   Amazon ECR
-   Azure Container Registry
-   Google Artifact Registry
-   Private registries

When an image is not available locally, Docker can pull it from a
registry.

``` text
Docker Host
     |
     | image not available
     v
Docker Registry
     |
     v
Docker Image
```

### Remember

> **Registry = Stores and distributes images**

------------------------------------------------------------------------

## 3.5 Docker Image

A **Docker Image** is a read-only package/template used to create
containers.

It contains the filesystem and application-related content required to
run the application.

For architecture, remember the relationship:

``` text
Docker Image
      ↓
Container Creation
      ↓
Docker Container
```

An image may come from:

``` text
Docker Registry
       ↓
   Docker Host
```

or already exist locally on the Docker Host.

### Remember

> **Image = Template used to create a container**

------------------------------------------------------------------------

## 3.6 containerd

**containerd** is a container runtime component responsible for managing
the **container lifecycle**.

Docker's daemon delegates container-related lifecycle operations to
containerd.

``` text
dockerd
   ↓
containerd
```

containerd then works with the low-level runtime, typically `runc`.

``` text
dockerd
   ↓
containerd
   ↓
runc
```

### Remember

> **containerd = Manages the container lifecycle**

------------------------------------------------------------------------

## 3.7 runc

**runc** is a low-level **OCI-compliant container runtime**.

It is responsible for creating and starting the actual container process
using the container configuration and Linux kernel features.

``` text
containerd
    ↓
  runc
    ↓
Linux Kernel
    ↓
Container Process
```

### Remember

> **runc = Low-level runtime that creates and starts the container
> process**

------------------------------------------------------------------------

## 3.8 Linux Kernel

Containers rely on the **Linux kernel of the Docker Host**.

The kernel provides mechanisms required for container isolation and
resource control, including:

-   **Namespaces** → isolation
-   **cgroups** → resource control
-   **Filesystem mechanisms** → filesystem isolation

``` text
runc
  ↓
Linux Kernel
  ↓
Container Process
```

### Remember

> **Containers share the host kernel rather than running a separate
> guest kernel.**

------------------------------------------------------------------------

## 3.9 Docker Container

A **Docker Container** is the runtime environment created from a Docker
image.

The application process runs inside this container.

``` text
Docker Image
     ↓
Container
     ↓
Application Process
```

From an architecture perspective, the container is the result of the
lower-level runtime operations performed through containerd and runc.

### Remember

> **Container = Runtime environment in which the application process
> runs**

------------------------------------------------------------------------

# 4. Docker Host Architecture

The **Docker Host** is the machine where Docker and its containers run.

A simplified Docker Host looks like this:

``` text
+------------------------------------------------+
|                 Docker Host                    |
|                                                |
|  Docker CLI                                    |
|       ↓                                        |
|  Docker Daemon (dockerd)                      |
|       ↓                                        |
|  containerd                                    |
|       ↓                                        |
|  runc                                          |
|       ↓                                        |
|  Linux Kernel                                  |
|       ↓                                        |
|  +----------------+  +----------------------+ |
|  | Container A    |  | Container B          | |
|  | Application A  |  | Application B        | |
|  +----------------+  +----------------------+ |
|                                                |
+------------------------------------------------+
```

The Docker Host provides:

-   CPU
-   Memory
-   Storage
-   Networking
-   Linux kernel
-   Docker runtime components

### Important

The **Docker CLI does not necessarily have to run on the same machine as
the Docker Daemon**. A client can communicate with a Docker daemon
remotely through the Docker API.

------------------------------------------------------------------------

# 5. Docker Client → Docker Daemon Communication

The basic communication flow is:

``` text
Developer
    ↓
Docker CLI
    ↓
Docker API
    ↓
Docker Daemon
```

For example:

``` bash
docker run nginx
```

The command does not mean that the CLI itself creates the container.

Instead:

1.  The user executes the command.
2.  Docker CLI interprets the command.
3.  CLI sends the request through the Docker API.
4.  `dockerd` receives the request.
5.  `dockerd` manages the required Docker operation.

### Interview point

> **The Docker CLI is the client; `dockerd` is the daemon that performs
> the Docker management operation.**

------------------------------------------------------------------------

# 6. What Happens During `docker run nginx`?

This is one of the most important Docker Architecture interview
questions.

Command:

``` bash
docker run nginx
```

High-level flow:

``` text
Docker CLI
    ↓
Docker API
    ↓
Docker Daemon (dockerd)
    ↓
Check local image
    ↓
Image available?
   / \
 Yes  No
  |    |
  |    v
  |  Docker Registry
  |    |
  |    v
  |  Docker Image
  |    |
  +----+
       ↓
   containerd
       ↓
      runc
       ↓
  Linux Kernel
       ↓
   Container
       ↓
Nginx Process
```

## Step 1 --- Docker CLI

The user runs:

``` bash
docker run nginx
```

The Docker CLI sends the request to the Docker Daemon through the Docker
API.

## Step 2 --- Docker Daemon

`dockerd` receives the request and determines that an `nginx` container
needs to be created and started.

## Step 3 --- Check the Image

Docker checks whether the required nginx image is available locally.

``` text
Is nginx image available?

       Yes
        ↓
   Use local image

       No
        ↓
 Pull from Registry
```

## Step 4 --- Pull Image if Required

If the image is unavailable locally, Docker pulls it from a configured
registry such as Docker Hub.

``` text
Docker Registry
      ↓
 Docker Image
      ↓
 Docker Host
```

## Step 5 --- containerd

Once the image is available, Docker delegates container lifecycle
operations to containerd.

``` text
dockerd
   ↓
containerd
```

## Step 6 --- runc

containerd works with the low-level OCI runtime, `runc`.

``` text
containerd
    ↓
   runc
```

runc creates and starts the container process according to the container
configuration.

## Step 7 --- Linux Kernel

runc uses Linux kernel capabilities to establish the container
environment.

Important kernel mechanisms include:

``` text
Namespaces → Isolation
cgroups    → Resource Control
```

## Step 8 --- Container and Application

The result is a running nginx container containing the nginx process.

``` text
Linux Kernel
     ↓
Nginx Container
     ↓
Nginx Process
```

------------------------------------------------------------------------

# 7. The Most Important Relationship: dockerd → containerd → runc

This relationship is frequently asked in Docker interviews.

``` text
Docker Daemon
    dockerd
       ↓
   containerd
       ↓
      runc
       ↓
 Linux Kernel
       ↓
Container Process
```

### dockerd

Responsible for the **Docker-level management**.

``` text
dockerd
→ receives Docker API requests
→ manages Docker resources
→ coordinates container operations
```

### containerd

Responsible for **container lifecycle management**.

``` text
containerd
→ manages container lifecycle
→ coordinates container runtime operations
```

### runc

Responsible for **low-level container creation and startup**.

``` text
runc
→ creates container
→ configures runtime environment
→ starts container process
```

### Easy Interview Answer

> **Docker Daemon manages the Docker operation, containerd manages the
> container lifecycle, and runc is the low-level OCI runtime that
> creates and starts the container process.**

------------------------------------------------------------------------

# 8. Docker Registry and Image Flow

A registry is involved when Docker needs to obtain an image that is not
available locally.

``` text
             Docker Registry
                    |
                    | Pull
                    v
              Docker Image
                    |
                    v
              Docker Host
                    |
                    v
                Container
```

### Local Image Scenario

``` text
Docker CLI
    ↓
dockerd
    ↓
Local Image
    ↓
containerd
    ↓
runc
    ↓
Container
```

### Image Not Available Locally

``` text
Docker CLI
    ↓
dockerd
    ↓
Docker Registry
    ↓
Docker Image
    ↓
containerd
    ↓
runc
    ↓
Container
```

------------------------------------------------------------------------

# 9. Docker Image vs Container in the Architecture

The image and container have different roles.

``` text
                  Docker Image
                       |
                 container creation
                       |
                       v
                Docker Container
                       |
                       v
                 Application
```

  Component              Role in Architecture
  ---------------------- ----------------------------------------
  **Docker Image**       Provides the packaged template/content
  **Docker Container**   Provides the runtime environment
  **Application**        Runs as a process inside the container

### Key Point

> An image is not the running application. The container is created from
> the image and provides the environment in which the application
> process runs.

------------------------------------------------------------------------

# 10. Linux Kernel: Namespaces and cgroups

Docker containers rely on Linux kernel features.

## Namespaces

Namespaces provide **isolation**.

They allow container processes to have an isolated view of resources
such as:

-   Processes
-   Network interfaces
-   Hostname
-   Mounts

``` text
Container A → Isolated view
Container B → Isolated view
```

### Remember

> **Namespaces = Isolation**

------------------------------------------------------------------------

## cgroups

Control groups, or **cgroups**, provide **resource control**.

They can control resources such as:

-   CPU
-   Memory
-   Number of processes

``` text
Container
    ↓
  cgroups
    ↓
CPU / Memory / Process limits
```

### Remember

> **cgroups = Resource Control**

------------------------------------------------------------------------

# 11. Complete Docker Architecture

The complete architecture can be visualized as:

``` text
                         Developer
                             |
                             v
                        Docker CLI
                             |
                             v
                        Docker API
                             |
                             v
                    Docker Daemon
                       (dockerd)
                             |
                    +--------+--------+
                    |                 |
                    | Image needed    | Image available
                    |                 |
                    v                 |
             Docker Registry          |
                    |                 |
                    v                 |
              Docker Image <----------+
                    |
                    v
                containerd
                    |
                    v
                  runc
                    |
                    v
              Linux Kernel
                    |
          +---------+---------+
          |                   |
     Namespaces            cgroups
      Isolation          Resource Control
          |                   |
          +---------+---------+
                    |
                    v
                Container
                    |
                    v
               Application
```

------------------------------------------------------------------------

# 12. Docker Architecture Interview Questions

## 1. What is Docker Architecture?

Docker uses a **client-server architecture**.

The Docker CLI communicates with the Docker Daemon through the Docker
API. The daemon manages Docker operations and works with containerd and
runc to create and start containers.

------------------------------------------------------------------------

## 2. What is the role of the Docker CLI?

The Docker CLI is the client interface used to send Docker commands.

``` text
CLI → Docker API → dockerd
```

It does not directly create the container.

------------------------------------------------------------------------

## 3. What is the Docker Daemon?

`dockerd` is the main Docker management service.

It receives Docker API requests and manages Docker resources and
container operations.

------------------------------------------------------------------------

## 4. What is a Docker Registry?

A Docker Registry stores and distributes Docker images.

If the required image is not available locally, Docker can pull it from
a registry.

------------------------------------------------------------------------

## 5. What happens when you run `docker run nginx`?

``` text
CLI
 ↓
Docker API
 ↓
dockerd
 ↓
Check local image
 ↓
Pull image if required
 ↓
containerd
 ↓
runc
 ↓
Linux Kernel
 ↓
Nginx Container
 ↓
Nginx Process
```

------------------------------------------------------------------------

## 6. What is containerd?

containerd is responsible for managing the **container lifecycle** and
coordinating container runtime operations.

------------------------------------------------------------------------

## 7. What is runc?

runc is a low-level **OCI-compliant container runtime** that creates and
starts the container process using Linux kernel capabilities.

------------------------------------------------------------------------

## 8. Why are both containerd and runc needed?

They operate at different levels.

``` text
containerd
   ↓
Lifecycle management

runc
   ↓
Low-level container creation/start
```

> **containerd manages; runc creates and starts.**

------------------------------------------------------------------------

## 9. Does Docker Registry create the container?

No.

The Registry only **stores and distributes images**.

The simplified execution flow is:

``` text
Registry
   ↓
Image
   ↓
containerd
   ↓
runc
   ↓
Container
```

------------------------------------------------------------------------

## 10. What is the role of the Linux Kernel in Docker Architecture?

The Linux kernel provides the underlying mechanisms used to isolate and
control container processes.

Two important concepts are:

``` text
Namespaces → Isolation
cgroups    → Resource Control
```

------------------------------------------------------------------------

# 13. Final Quick Revision

## Architecture Flow

``` text
docker run nginx
      ↓
Docker CLI
      ↓
Docker API
      ↓
Docker Daemon
      ↓
Docker Registry (if required)
      ↓
Docker Image
      ↓
containerd
      ↓
runc
      ↓
Linux Kernel
      ↓
Docker Container
      ↓
Application
```

## Core Components

  Component          One-line responsibility
  ------------------ -----------------------------------------
  **Docker CLI**     Sends Docker commands
  **Docker API**     Connects client and daemon
  **dockerd**        Manages Docker operations
  **Registry**       Stores/distributes images
  **Image**          Template for creating containers
  **containerd**     Manages container lifecycle
  **runc**           Creates/starts the container process
  **Linux Kernel**   Provides isolation and resource control
  **Container**      Runtime environment for the application

## Most Important Flow

``` text
Docker CLI
    ↓
Docker API
    ↓
dockerd
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
    ↓
Container
    ↓
Application
```

## Final Memory Trick

> **"CLI gives the command → Docker API carries the request → dockerd
> manages it → Registry provides the image → containerd manages the
> lifecycle → runc creates and starts the container → Linux Kernel
> provides isolation → Container runs the application."**
