# Production-Ready Dockerfile — Top 10 Instructions

<img width="549" height="833" alt="image" src="https://github.com/user-attachments/assets/46bad1c8-ebe9-4a48-bf0a-0500295c3aac" />


## Quick Study Guide

A production-ready Dockerfile should be **small, secure, predictable, and easy to maintain**.

---

## 1. FROM

Defines the base image.

```dockerfile
FROM python:3.12-slim
```

**Production tip:** Prefer minimal, trusted base images when possible. Smaller images generally reduce attack surface and image size.

---

## 2. WORKDIR

Sets the working directory inside the container.

```dockerfile
WORKDIR /app
```

**Production tip:** Use `WORKDIR` instead of repeatedly using absolute paths.

---

## 3. COPY

Copies files from the Docker build context into the image.

```dockerfile
COPY requirements.txt .
COPY . .
```

**Production tip:** Copy only what the application needs. Use `.dockerignore` to exclude unnecessary files.

---

## 4. RUN

Executes commands while the image is being built.

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt     && rm -rf /root/.cache
```

**Production tip:** Combine related commands and clean unnecessary caches in the same layer where practical.

---

## 5. ENV

Defines environment variables available in the container environment.

```dockerfile
ENV APP_ENV=production
```

**Important:** Do not hardcode passwords, API keys, tokens, or other secrets in a Dockerfile.

---

## 6. ARG

Defines a build-time variable.

```dockerfile
ARG APP_VERSION=1.0
```

Build example:

```bash
docker build --build-arg APP_VERSION=2.0 .
```

**Remember:**

- `ARG` → mainly build time
- `ENV` → available in the container environment

---

## 7. EXPOSE

Documents the port on which the application listens.

```dockerfile
EXPOSE 8080
```

**Important:** `EXPOSE` does **not** publish the port to the host.

Example:

```bash
docker run -p 8080:8080 myapp
```

---

## 8. USER

Specifies which user runs the application.

```dockerfile
RUN useradd --create-home appuser     && chown -R appuser:appuser /app

USER appuser
```

**Production tip:** Avoid running applications as `root` unless there is a specific requirement. A dedicated non-root user follows the principle of least privilege.

---

## 9. ENTRYPOINT

Defines the main executable for the container.

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

Think:

> **What is this container supposed to run?**

---

## 10. CMD

Provides the default command or default arguments.

```dockerfile
CMD ["--port", "8080"]
```

A common pattern is:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```

Think:

- `ENTRYPOINT` → main executable
- `CMD` → default arguments / default command

---

# Complete Production-Oriented Example

```dockerfile
# 1. FROM
FROM python:3.12-slim

# 2. WORKDIR
WORKDIR /app

# 6. ARG
ARG APP_VERSION=1.0

# 5. ENV
ENV APP_ENV=production

# 3. COPY
COPY requirements.txt .

# 4. RUN
RUN pip install --no-cache-dir -r requirements.txt     && rm -rf /root/.cache

# Copy application files
COPY . .

# 8. USER
RUN useradd --create-home appuser     && chown -R appuser:appuser /app

USER appuser

# 7. EXPOSE
EXPOSE 8080

# 9. ENTRYPOINT
ENTRYPOINT ["python", "app.py"]

# 10. CMD
CMD ["--port", "8080"]
```

---

# Bonus: .dockerignore

`.dockerignore` prevents unnecessary files from being sent as part of the Docker build context.

```text
.git
.gitignore
__pycache__
*.pyc
.env
.env.*
logs/
*.log
.vscode/
.idea/
```

**Good practice:** Never send unnecessary files, local dependencies, logs, or secrets into the build context.

---

# Quick Revision

| Instruction | Main Purpose |
|---|---|
| `FROM` | Select base image |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files into image |
| `RUN` | Execute build-time commands |
| `ENV` | Set environment variables |
| `ARG` | Define build-time variables |
| `EXPOSE` | Document application port |
| `USER` | Run as specified user |
| `ENTRYPOINT` | Define main executable |
| `CMD` | Define defaults / arguments |

---

# Production Checklist

Before using a Dockerfile in production, ask:

- [ ] Am I using a minimal and trusted base image?
- [ ] Did I set a clear `WORKDIR`?
- [ ] Am I copying only required files?
- [ ] Did I use `.dockerignore`?
- [ ] Did I clean unnecessary package caches?
- [ ] Are secrets kept outside the Dockerfile?
- [ ] Did I use `ARG` and `ENV` correctly?
- [ ] Did I understand that `EXPOSE` does not publish ports?
- [ ] Is the application running as a non-root user?
- [ ] Did I correctly use `ENTRYPOINT` and `CMD`?

> **Key takeaway:** Don't just memorize Dockerfile instructions. Understand why and when to use each instruction, especially when building images for production.
