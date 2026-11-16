# Docker Stop vs Docker Kill — Study Notes

## What is the difference?

Both commands stop a running Docker container, but they differ in **shutdown behavior, signals, and production usage**.

| Feature | `docker stop` | `docker kill` |
|---|---|---|
| Shutdown | Attempts graceful shutdown | Immediate signal delivery |
| Default signal | Container's configured stop signal (usually `SIGTERM`), followed by `SIGKILL` if needed | `SIGKILL` |
| Grace period | Yes; configurable | No normal grace period with default `SIGKILL` |
| Best use | Normal application shutdown | Stuck containers or urgent termination |

## Practical example: Java application

Assume a Java application is running in a container named `my-app`.

**1. Check the running container**

```bash
docker ps
```

**2. Stop gracefully (allow up to 30 seconds)**

```bash
docker stop --time 30 my-app
```

Docker sends the configured stop signal (normally `SIGTERM`). If the container has not exited after 30 seconds, Docker sends `SIGKILL`.

**3. Restart the container for the next example**

```bash
docker start my-app
```

**4. Force immediate termination**

```bash
docker kill my-app
```

By default, this sends `SIGKILL` without a graceful shutdown period. You can send a different signal using `--signal`, for example `docker kill --signal=SIGTERM my-app`.

**5. Verify the container is no longer running**

```bash
docker ps
# To view stopped containers too:
docker ps -a
```

## Production best practices

- **Prefer `docker stop`** so the application can finish requests, close connections, and release resources.
- Ensure the application handles the stop signal correctly. In containerized Java apps, the Java process should receive termination signals (for example, by using an appropriate exec-form entrypoint).
- Choose a grace period that matches the application's shutdown needs.
- Use **`docker kill` only when necessary**, as forced termination can interrupt in-flight requests or leave work incomplete.

**Interview takeaway:** `docker stop` normally sends `SIGTERM`, waits for a grace period, and then uses `SIGKILL` if needed; `docker kill` sends `SIGKILL` immediately by default.
