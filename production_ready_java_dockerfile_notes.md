# Production-Ready Java Dockerfile --- Interview & Production Notes

## 1. Production Goal

A production Java container should not only run the application. It
should also be:

-   Small and efficient
-   Secure
-   Reproducible
-   Cache-friendly
-   Free from secrets
-   Easy to operate in Kubernetes or another container platform
-   Built with only the runtime components required to execute the
    application

A good production pattern is:

``` text
Source Code
    ↓
Multi-Stage Build
    ↓
JDK + Maven/Gradle Build
    ↓
Application JAR
    ↓
Minimal JRE Runtime Image
    ↓
Non-Root User
    ↓
ENTRYPOINT
    ↓
Java Application
```

------------------------------------------------------------------------

# 2. Production Dockerfile

``` dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw \
    && ./mvnw -B dependency:go-offline

COPY src ./src

RUN ./mvnw -B clean package -DskipTests \
    && mv target/*.jar target/app.jar


FROM eclipse-temurin:21-jre

WORKDIR /app

RUN groupadd --system appgroup \
    && useradd --system \
       --gid appgroup \
       --no-create-home \
       appuser

COPY --from=builder /build/target/app.jar /app/app.jar

RUN chown appuser:appgroup /app/app.jar

USER appuser

EXPOSE 8080

ENV JAVA_OPTS=""

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
```

> Note: The exact base image should be selected according to the
> organization's approved Java distribution, security policy, patching
> strategy, and supported Java version.

------------------------------------------------------------------------

# 3. Multi-Stage Build

## Why?

The build environment requires tools such as:

-   JDK
-   Maven/Gradle
-   Maven dependencies
-   Source code
-   Build plugins

The production container normally needs only:

-   Java runtime
-   Application JAR
-   Required runtime files

Therefore, use two stages.

``` dockerfile
FROM eclipse-temurin:21-jdk AS builder
```

This stage builds the application.

``` dockerfile
FROM eclipse-temurin:21-jre
```

This stage runs the application.

The final image does not contain Maven, source code, or the JDK build
environment.

### Production benefit

``` text
Build Image
JDK + Maven + Source + Dependencies
              ↓
           JAR file
              ↓
Runtime Image
JRE + JAR
```

This reduces unnecessary components and helps reduce the attack surface.

------------------------------------------------------------------------

# 4. Docker Layer Caching

The order of `COPY` instructions matters.

Instead of copying the entire application first:

``` dockerfile
COPY . .
```

copy dependency-related files first:

``` dockerfile
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw \
    && ./mvnw -B dependency:go-offline
```

Then copy application source:

``` dockerfile
COPY src ./src
```

### Why?

Docker builds images layer by layer.

If the Java source changes but `pom.xml` has not changed, Docker can
reuse the dependency layer.

``` text
pom.xml unchanged
      ↓
Dependency layer cached
      ↓
Source code changes
      ↓
Only later layers rebuild
```

This can significantly improve CI/CD build times.

------------------------------------------------------------------------

# 5. JDK vs JRE

The build stage needs the JDK:

``` dockerfile
FROM eclipse-temurin:21-jdk AS builder
```

because compilation requires development tools.

The runtime stage can use a JRE/runtime image:

``` dockerfile
FROM eclipse-temurin:21-jre
```

because the application has already been compiled into a JAR.

### Production principle

> Build with the JDK. Run with the smallest appropriate runtime.

Do not automatically use a large JDK image for the final production
container when the application only needs a runtime.

------------------------------------------------------------------------

# 6. Non-Root Container

By default, avoid running application processes as root.

Create a dedicated user:

``` dockerfile
RUN groupadd --system appgroup \
    && useradd --system \
       --gid appgroup \
       --no-create-home \
       appuser
```

Then:

``` dockerfile
USER appuser
```

The application therefore runs as:

``` text
appuser
    ↓
Java Process
    ↓
Container
```

instead of:

``` text
root
    ↓
Java Process
```

### Why?

If an application is compromised, running as a non-root user can reduce
the potential impact inside the container.

This is an important container-security practice.

------------------------------------------------------------------------

# 7. Copy Only the Runtime Artifact

The runtime stage uses:

``` dockerfile
COPY --from=builder /build/target/app.jar /app/app.jar
```

Only the application artifact is copied from the builder stage.

The final image should not contain:

-   Maven
-   Source code
-   `.git`
-   Build caches
-   IDE configuration
-   Compiler
-   Test artifacts
-   Unnecessary development tools

### Principle

> The final image should contain only what is required to run the
> application.

------------------------------------------------------------------------

# 8. ENTRYPOINT

For a container whose primary responsibility is running one Java
application:

``` dockerfile
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
```

`ENTRYPOINT` defines the main process of the container.

The important part is:

``` text
exec java ...
```

`exec` replaces the shell process with the Java process.

This helps Java become the main process and receive termination signals
properly.

This matters when the application runs under Kubernetes or another
orchestrator and receives signals such as `SIGTERM` during shutdown.

### Why not use shell-form ENTRYPOINT?

Avoid:

``` dockerfile
ENTRYPOINT java -jar /app/app.jar
```

Prefer exec-style configuration where practical.

------------------------------------------------------------------------

# 9. JVM Configuration

Avoid hardcoding environment-specific JVM values directly into the
image.

For example:

``` dockerfile
ENV JAVA_OPTS=""
```

The deployment platform can provide values appropriate for the
environment.

Example:

``` text
Development → smaller memory settings
QA          → test environment settings
Production  → production resource settings
```

For Kubernetes, JVM options can be provided through environment
variables or deployment configuration.

Modern JVMs are container-aware, but memory configuration should still
be aligned with the CPU and memory limits assigned to the workload.

Avoid blindly setting a very large:

``` text
-Xmx
```

without considering the container's memory limit.

------------------------------------------------------------------------

# 10. EXPOSE

``` dockerfile
EXPOSE 8080
```

`EXPOSE` documents the port the application listens on.

It does not publish the port by itself.

For example:

``` text
EXPOSE 8080
```

does not mean the application is automatically reachable from the
internet.

The actual networking behavior is controlled by Docker runtime
configuration or the orchestration platform.

------------------------------------------------------------------------

# 11. .dockerignore

A production Docker build should also have a `.dockerignore`.

Example:

``` dockerignore
.git
.gitignore
.idea
.vscode
*.iml

target/
build/
*.class

*.log
logs/

.env
.env.*
*.pem
*.key

Dockerfile
docker-compose*.yml

README.md
docs/

node_modules/
.DS_Store
```

### Why?

Docker sends the build context to the Docker daemon/build system.

There is no reason to send unnecessary files such as:

-   Git metadata
-   IDE configuration
-   Local build output
-   Logs
-   Local environment files
-   Private keys
-   Documentation
-   `node_modules`

A smaller build context can improve build efficiency and reduces the
chance of accidentally copying unwanted files.

------------------------------------------------------------------------

# 12. Secrets --- Never Bake Them Into the Image

Never do this:

``` dockerfile
ENV DB_PASSWORD="mypassword"
```

or:

``` dockerfile
COPY .env /app/.env
```

or:

``` dockerfile
ENV AWS_SECRET_ACCESS_KEY="..."
```

Docker images can be stored in registries and accessed by multiple
systems and users.

Secrets should be supplied at runtime using mechanisms such as:

-   Kubernetes Secrets
-   Cloud secret managers
-   Vault
-   CI/CD secret stores
-   Platform-specific secret mechanisms

### Production principle

> Build once. Inject environment-specific configuration and secrets at
> runtime.

------------------------------------------------------------------------

# 13. Image Versioning

Avoid blindly relying on:

``` dockerfile
FROM eclipse-temurin:latest
```

A moving `latest` tag can change over time.

Prefer an explicit, organization-approved version/tag strategy.

For example:

``` dockerfile
FROM eclipse-temurin:21-jdk AS builder
FROM eclipse-temurin:21-jre
```

For stricter reproducibility, organizations may pin image digests in
addition to using controlled tags.

The exact strategy should align with the organization's patching and
image-update process.

------------------------------------------------------------------------

# 14. Image Security Scanning

A production image should be scanned before deployment.

Typical checks include:

-   OS vulnerabilities
-   Java/runtime vulnerabilities
-   Application dependency vulnerabilities
-   Secrets accidentally included in the image
-   Misconfiguration
-   Unsupported packages

Common tools include:

-   Trivy
-   Docker Scout
-   Cloud/container registry scanning
-   Enterprise container-security platforms

Typical CI/CD flow:

``` text
Build
  ↓
Test
  ↓
Build Image
  ↓
Scan Image
  ↓
Push Registry
  ↓
Deploy
```

Do not treat a successful `docker build` as proof that the image is
production-ready.

------------------------------------------------------------------------

# 15. Health Checks in Production

For a Java/Spring Boot application, health should normally be designed
at the application/orchestrator level.

For Kubernetes, use:

-   `startupProbe`
-   `readinessProbe`
-   `livenessProbe`

Example conceptual flow:

``` text
Container starts
      ↓
Startup Probe
      ↓
Application initialized
      ↓
Readiness Probe
      ↓
Traffic allowed
      ↓
Liveness Probe
      ↓
Detect unhealthy application
```

Do not add a Docker `HEALTHCHECK` simply because it is a production
image.

Choose the health-check mechanism based on the deployment platform.

------------------------------------------------------------------------

# 16. Common Mistakes

## Mistake 1 --- One huge image

``` dockerfile
FROM eclipse-temurin:21-jdk
```

and keeping the complete build environment in production.

### Better

Use a multi-stage build and a runtime image.

------------------------------------------------------------------------

## Mistake 2 --- Running as root

``` dockerfile
USER root
```

### Better

Create and use a dedicated application user.

------------------------------------------------------------------------

## Mistake 3 --- Copying everything

``` dockerfile
COPY . .
```

This can send unnecessary files into the build context and can make
caching less efficient.

### Better

Use targeted `COPY` instructions and `.dockerignore`.

------------------------------------------------------------------------

## Mistake 4 --- Baking secrets into the image

``` dockerfile
ENV PASSWORD=...
```

### Better

Inject secrets at runtime.

------------------------------------------------------------------------

## Mistake 5 --- Using `latest`

``` dockerfile
FROM eclipse-temurin:latest
```

### Better

Use a deliberate versioning and patching strategy.

------------------------------------------------------------------------

## Mistake 6 --- Installing unnecessary tools in runtime

Avoid adding:

-   Maven
-   Git
-   Compilers
-   Debugging utilities
-   Build caches

unless there is a documented runtime requirement.

------------------------------------------------------------------------

## Mistake 7 --- Ignoring JVM/container resources

Do not configure JVM memory independently of the container's memory
limits.

The JVM configuration should be designed together with:

``` text
Container Memory
        +
JVM Heap
        +
Off-Heap Memory
        +
Native Memory
```

------------------------------------------------------------------------

# 17. Production Checklist

Before deploying a Java container, verify:

-   [ ] Multi-stage build used
-   [ ] JDK used only where required
-   [ ] Runtime image is minimal
-   [ ] Java version is deliberately selected
-   [ ] Dependency layer is cache-friendly
-   [ ] `.dockerignore` exists
-   [ ] Source/build tools are excluded from runtime image
-   [ ] Container does not run as root
-   [ ] Secrets are not inside Dockerfile/image
-   [ ] `ENTRYPOINT` uses appropriate exec/signal handling
-   [ ] JVM settings match container resources
-   [ ] Application health checks are configured at the platform level
-   [ ] Image vulnerability scanning is part of CI/CD
-   [ ] Image is tagged/versioned appropriately
-   [ ] Runtime image contains only required components

------------------------------------------------------------------------

# 18. Interview Questions

### Q1. Why use multi-stage Docker builds for Java?

To separate the build environment from the runtime environment and keep
build tools out of the final production image.

### Q2. Why copy `pom.xml` before `src`?

To maximize Docker layer caching. Dependency layers can be reused when
only source code changes.

### Q3. Why use JRE instead of JDK in the runtime stage?

The application has already been compiled, so the runtime normally does
not require Java development tools.

### Q4. Why run the container as a non-root user?

To reduce the potential impact of a container/application compromise and
follow least-privilege principles.

### Q5. Why use `ENTRYPOINT`?

It defines the primary process the container is intended to run. For a
Java application container, Java is the main executable.

### Q6. Why is `exec` important when using a shell?

It replaces the shell process with Java, helping Java become the main
process and receive termination signals correctly.

### Q7. Does `EXPOSE 8080` publish port 8080?

No. `EXPOSE` documents the intended container port. Port publishing is
handled by Docker runtime configuration or the orchestration platform.

### Q8. Why use `.dockerignore`?

To keep unnecessary files out of the build context and reduce build
overhead and accidental inclusion of unwanted files.

### Q9. Where should secrets be stored?

Outside the image, using runtime secret-management mechanisms.

### Q10. Is a successful Docker build enough for production?

No. Production readiness also requires security scanning, vulnerability
management, runtime configuration, resource management, health checks,
logging/observability, and an appropriate deployment strategy.

------------------------------------------------------------------------

# 19. Production Mental Model

When reviewing any Java Dockerfile, think in this order:

``` text
1. Can I build it efficiently?
            ↓
2. Can I make the final image smaller?
            ↓
3. Can I remove unnecessary runtime components?
            ↓
4. Does it run as non-root?
            ↓
5. Are secrets outside the image?
            ↓
6. Does Java receive termination signals correctly?
            ↓
7. Is JVM memory aligned with container limits?
            ↓
8. Can I scan and patch the image?
            ↓
9. Can the same image move across environments?
            ↓
10. Can the orchestrator monitor and safely operate it?
```

## Key takeaway

A production Dockerfile is not just about:

``` text
FROM
COPY
RUN
ENTRYPOINT
```

It is about designing a **secure, minimal, reproducible, cache-efficient
and operationally safe runtime artifact**.

> **Build with the tools you need. Run with only the tools you need.
> Configure environments at runtime. Keep secrets out of images. Run as
> non-root. Scan everything before deployment.**
