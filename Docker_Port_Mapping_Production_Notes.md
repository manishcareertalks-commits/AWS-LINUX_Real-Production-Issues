# Docker Port Mapping / Port Forwarding --- Production Notes

**Interview question:** You have hosted an application inside a Docker
container. How do you access it from the internet?

## 1. How port mapping works

A container has its own network namespace. To make an application
reachable through the Docker host, publish a **host port** to the
**container port**.

**Traffic flow:** Client → (HTTPS) AWS Application Load Balancer → EC2
host port `8080` → Docker container port `8080` → Application

The application must listen on `0.0.0.0:8080` inside the container, not
just `127.0.0.1:8080`.

For a Spring Boot application (`application.properties`):

``` properties
server.port=8080
server.address=0.0.0.0
```

## 2. Build and run the container

Example `Dockerfile`:

``` dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build and publish port 8080:

``` bash
docker build -t my-app:1.0.0 .
docker run -d --name my-app -p 8080:8080 my-app:1.0.0
docker ps
docker port my-app
```

**Syntax:** `-p HOST_PORT:CONTAINER_PORT`. For example, `-p 9090:8080`
sends host port 9090 to container port 8080.

**Important:** `EXPOSE 8080` only documents the intended container port.
It **does not publish** that port. Publishing is configured at container
creation (`-p`, Compose, or an orchestrator). `docker run -P` publishes
exposed ports to automatically allocated host ports.

## 3. AWS EC2 production setup

1.  Run Docker containers on EC2 instances, preferably in **private
    subnets**.
2.  Put an **internet-facing Application Load Balancer** in public
    subnets.
3.  Configure an **HTTPS listener on port 443** with an ACM certificate.
4.  Configure an ALB target group with the appropriate EC2/container
    target and port (e.g., HTTP `8080`), plus health checks.
5.  Allow inbound **443** to the ALB security group from intended
    clients.
6.  Allow inbound **8080** to EC2 **only from the ALB security group**;
    do not open the application port to the entire internet.
7.  Point the application domain to the ALB using Route 53 (optional WAF
    for additional protection).

**Note:** ECS is optional if you manage Docker directly on EC2. With
ECS, task networking and target-group configuration depend on the
selected network mode.

## 4. Quick interview revision

  -----------------------------------------------------------------------
  Concept                             What to remember
  ----------------------------------- -----------------------------------
  `0.0.0.0`                           App listens on all container
                                      interfaces

  `127.0.0.1`                         App listens only on container
                                      loopback

  `EXPOSE 8080`                       Documents port; does **not**
                                      publish it

  `-p 8080:8080`                      Publishes host port 8080 to
                                      container port 8080

  `-p 127.0.0.1:8080:8080`            Binds published port to host
                                      loopback only

  Security group                      Permit only required sources and
                                      ports

  ALB + HTTPS                         Public TLS entry point, routing,
                                      and health checks

  Production                          Private EC2, restricted SGs, TLS,
                                      health checks, scaling
  -----------------------------------------------------------------------

**Interview-ready answer:** "First, I ensure the application listens on
`0.0.0.0`. Then I publish its container port using
`docker run -p 8080:8080`. On AWS, I put an HTTPS Application Load
Balancer in front of private EC2 instances, allow port 8080 only from
the ALB security group, configure health checks, and avoid directly
exposing application ports to the internet."

------------------------------------------------------------------------

**ManishCareerTalks** · Cloud \| DevOps \| AI \| Career Growth
