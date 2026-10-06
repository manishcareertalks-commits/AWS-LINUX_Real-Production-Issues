# Docker CMD vs ENTRYPOINT

## 1. Overview

Both `CMD` and `ENTRYPOINT` define what happens when a Docker container starts.

The key difference:

- **ENTRYPOINT** → Defines the main application/process that should run.
- **CMD** → Defines the default command or default arguments.

A common production pattern is:

> **ENTRYPOINT = Fixed application**
>
> **CMD = Default configurable arguments**

---

# 2. Production-Style Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

### What each instruction does

| Instruction | Purpose |
|---|---|
| `FROM` | Uses Java 21 JRE as the base image |
| `WORKDIR` | Sets `/app` as the working directory |
| `COPY` | Copies the Java application into the image |
| `ENTRYPOINT` | Defines the main application |
| `CMD` | Defines the default application argument |

---

# 3. Understanding ENTRYPOINT

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

`ENTRYPOINT` defines the **main application/process** of the container.

The flow is:

```text
Container starts
      ↓
java -jar app.jar
      ↓
Spring Boot application starts
```

The application itself is fixed by the `ENTRYPOINT`.

---

# 4. Understanding CMD

```dockerfile
CMD ["--spring.profiles.active=prod"]
```

`CMD` provides the **default argument** to the application.

Therefore, the default command becomes:

```bash
java -jar app.jar --spring.profiles.active=prod
```

The `prod` profile is the default.

---

# 5. What Happens with docker run?

When we execute:

```bash
docker run myapp
```

Docker combines:

```text
ENTRYPOINT
    +
CMD
```

Result:

```bash
java -jar app.jar --spring.profiles.active=prod
```

Conceptually:

```text
ENTRYPOINT
java -jar app.jar

        +

CMD
--spring.profiles.active=prod

        ↓

java -jar app.jar --spring.profiles.active=prod
```

---

# 6. Overriding CMD

Suppose we want to run the application using the `dev` Spring profile.

```bash
docker run myapp --spring.profiles.active=dev
```

The supplied argument replaces the default `CMD`.

Docker effectively runs:

```bash
java -jar app.jar --spring.profiles.active=dev
```

The important point:

```text
ENTRYPOINT → remains
CMD        → replaced
```

Therefore:

> **CMD can easily be overridden at runtime.**

---

# 7. Overriding ENTRYPOINT

Suppose we want to completely replace the Java application.

We can explicitly use:

```bash
docker run --entrypoint bash myapp
```

Now:

```text
ENTRYPOINT
java -jar app.jar

        ↓

replaced by

        ↓

bash
```

The container uses:

```bash
bash
```

as its entrypoint.

Therefore:

> **ENTRYPOINT can also be overridden, but it requires the explicit `--entrypoint` option.**

---

# 8. CMD vs ENTRYPOINT

| Feature | CMD | ENTRYPOINT |
|---|---|---|
| Main purpose | Default command/arguments | Main application/process |
| Runtime arguments override it? | Yes | No |
| Explicit override | Runtime command/arguments | `--entrypoint` |
| Common production use | Default runtime options | Fixed application |
| Example | `--spring.profiles.active=prod` | `java -jar app.jar` |

---

# 9. Important Interview Point

A common interview statement is:

> **CMD can be overridden at runtime by providing a command/arguments after the image name. ENTRYPOINT is not replaced by those arguments; it is explicitly overridden using `--entrypoint`.**

Example:

```bash
docker run myapp --spring.profiles.active=dev
```

Result:

```bash
java -jar app.jar --spring.profiles.active=dev
```

Whereas:

```bash
docker run --entrypoint bash myapp
```

Result:

```bash
bash
```

---

# 10. Why Use ENTRYPOINT + CMD Together?

This combination is useful when you want to:

- Fix the application that should run.
- Allow runtime configuration.
- Reuse the same image across environments.
- Provide sensible default options.
- Override those options when required.

Example:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

Think of it as:

```text
ENTRYPOINT
"What should always run?"

        ↓

java -jar app.jar


CMD
"What should be the default options?"

        ↓

--spring.profiles.active=prod
```

---

# 11. Production Example

The same Docker image can be used for different environments.

## Production

```bash
docker run myapp
```

Runs:

```bash
java -jar app.jar --spring.profiles.active=prod
```

## Development

```bash
docker run myapp --spring.profiles.active=dev
```

Runs:

```bash
java -jar app.jar --spring.profiles.active=dev
```

## Testing

```bash
docker run myapp --spring.profiles.active=test
```

Runs:

```bash
java -jar app.jar --spring.profiles.active=test
```

The Docker image remains the same.

Only the runtime argument changes.

---

# 12. Exec Form vs Shell Form

Docker supports two common forms.

## Exec Form

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Shell Form

```dockerfile
ENTRYPOINT java -jar app.jar
```

For production containers, the **exec form is generally preferred**.

It allows the application to run as the container's main process more directly and generally provides better signal handling.

The same applies to `CMD`.

### Exec Form

```dockerfile
CMD ["--spring.profiles.active=prod"]
```

### Shell Form

```dockerfile
CMD --spring.profiles.active=prod
```

---

# 13. CMD Alone

You can use `CMD` without `ENTRYPOINT`.

Example:

```dockerfile
FROM ubuntu:24.04

CMD ["echo", "Hello Docker"]
```

Running:

```bash
docker run myimage
```

runs:

```bash
echo Hello Docker
```

But:

```bash
docker run myimage ls
```

replaces the `CMD` and runs:

```bash
ls
```

This demonstrates why `CMD` is considered a default command.

---

# 14. ENTRYPOINT Alone

You can also use `ENTRYPOINT` without `CMD`.

Example:

```dockerfile
FROM ubuntu:24.04

ENTRYPOINT ["echo"]
```

Running:

```bash
docker run myimage Hello
```

results in:

```bash
echo Hello
```

The argument supplied after the image name is passed to the `ENTRYPOINT`.

---

# 15. ENTRYPOINT + CMD Together

Example:

```dockerfile
FROM ubuntu:24.04

ENTRYPOINT ["echo"]

CMD ["Hello Docker"]
```

Running:

```bash
docker run myimage
```

results in:

```bash
echo Hello Docker
```

Running:

```bash
docker run myimage "Hello DevOps"
```

results in:

```bash
echo Hello DevOps
```

Here:

```text
ENTRYPOINT → echo
CMD        → default argument
```

---

# 16. Docker Command Behavior

## Case 1: CMD Only

Dockerfile:

```dockerfile
CMD ["echo", "Hello Docker"]
```

Run:

```bash
docker run myimage
```

Result:

```bash
echo Hello Docker
```

Override:

```bash
docker run myimage "Hello DevOps"
```

Result:

```bash
Hello DevOps
```

The original `CMD` is replaced.

---

## Case 2: ENTRYPOINT Only

Dockerfile:

```dockerfile
ENTRYPOINT ["echo"]
```

Run:

```bash
docker run myimage "Hello Docker"
```

Result:

```bash
echo Hello Docker
```

The argument is appended to the `ENTRYPOINT`.

---

## Case 3: ENTRYPOINT + CMD

Dockerfile:

```dockerfile
ENTRYPOINT ["echo"]

CMD ["Hello Docker"]
```

Run:

```bash
docker run myimage
```

Result:

```bash
echo Hello Docker
```

Override:

```bash
docker run myimage "Hello DevOps"
```

Result:

```bash
echo Hello DevOps
```

---

# 17. Mental Model

A useful way to understand the combination is:

```text
ENTRYPOINT = executable
CMD        = default arguments
```

For example:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

Think:

```text
Executable
    ↓
java -jar app.jar

Default arguments
    ↓
--spring.profiles.active=prod
```

Together:

```bash
java -jar app.jar --spring.profiles.active=prod
```

If runtime arguments are supplied:

```bash
docker run myapp --spring.profiles.active=dev
```

Then:

```text
ENTRYPOINT
java -jar app.jar

        +

Runtime arguments
--spring.profiles.active=dev
```

Result:

```bash
java -jar app.jar --spring.profiles.active=dev
```

---

# 18. Common Interview Questions

## Q1. What is CMD in Docker?

`CMD` specifies the default command or default arguments that should be executed when a container starts.

It can be overridden by providing a command or arguments during `docker run`.

---

## Q2. What is ENTRYPOINT?

`ENTRYPOINT` specifies the main executable or application that the container is intended to run.

It is not replaced by normal arguments supplied after the image name.

To explicitly replace it:

```bash
docker run --entrypoint <command> <image>
```

---

## Q3. Can CMD and ENTRYPOINT be used together?

Yes.

Example:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

Here:

```text
ENTRYPOINT → application
CMD        → default arguments
```

---

## Q4. What happens if I run this?

```bash
docker run myapp --spring.profiles.active=dev
```

With:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

Docker runs:

```bash
java -jar app.jar --spring.profiles.active=dev
```

The `CMD` is replaced.

---

## Q5. How do you override ENTRYPOINT?

Use:

```bash
docker run --entrypoint bash myapp
```

This replaces the Dockerfile's `ENTRYPOINT`.

---

## Q6. Which is preferred for production?

A common production pattern is:

```dockerfile
ENTRYPOINT ["application"]

CMD ["default", "arguments"]
```

This makes the application fixed while keeping default runtime options configurable.

---

# 19. Common Mistakes

## Mistake 1: Thinking CMD always runs

Not necessarily.

If runtime arguments replace the `CMD`, the original `CMD` will not be used.

---

## Mistake 2: Thinking arguments replace ENTRYPOINT

For example:

```bash
docker run myapp bash
```

does **not** automatically replace:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Instead, `bash` is treated as an argument to the entrypoint.

The effective command would conceptually become:

```bash
java -jar app.jar bash
```

To replace the entrypoint:

```bash
docker run --entrypoint bash myapp
```

---

## Mistake 3: Using shell form without understanding signal handling

Instead of:

```dockerfile
ENTRYPOINT java -jar app.jar
```

prefer:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

for production workloads where direct process execution and signal handling matter.

---

# 20. Production Perspective

When designing a production Docker image, ask:

### Question 1

**What application should always run?**

Put that in:

```dockerfile
ENTRYPOINT
```

Example:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Question 2

**What should be configurable by default?**

Put that in:

```dockerfile
CMD
```

Example:

```dockerfile
CMD ["--spring.profiles.active=prod"]
```

This gives you:

```text
Fixed application
       +
Configurable defaults
       ↓
Reusable Docker image
```

---

# 21. Important Production Note

Although using Spring profiles as a `CMD` example is useful for understanding Docker behavior, in real production environments you should carefully consider how configuration and secrets are managed.

For example:

- Environment variables
- Kubernetes ConfigMaps
- Kubernetes Secrets
- External configuration systems
- Secret managers

Avoid baking sensitive values directly into the Docker image.

---

# 22. Quick Revision

```text
ENTRYPOINT = Main application / executable
CMD        = Default command / arguments
```

Example:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

### Default

```bash
docker run myapp
```

↓

```bash
java -jar app.jar --spring.profiles.active=prod
```

### Override CMD

```bash
docker run myapp --spring.profiles.active=dev
```

↓

```bash
java -jar app.jar --spring.profiles.active=dev
```

### Override ENTRYPOINT

```bash
docker run --entrypoint bash myapp
```

↓

```bash
bash
```

---

# 23. One-Line Interview Answer

> **ENTRYPOINT defines the main application or executable that the container is intended to run, while CMD provides the default command or arguments. Runtime arguments replace CMD, while replacing ENTRYPOINT requires the explicit `--entrypoint` option.**

---

# 24. Final Rule to Remember

```text
ENTRYPOINT → What should run?
CMD        → What should be the default arguments?
```

### Production Pattern

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]

CMD ["--spring.profiles.active=prod"]
```

### Remember

> **ENTRYPOINT = Fixed application**
>
> **CMD = Default configurable arguments**
>
> **Runtime arguments override CMD**
>
> **`--entrypoint` overrides ENTRYPOINT**
