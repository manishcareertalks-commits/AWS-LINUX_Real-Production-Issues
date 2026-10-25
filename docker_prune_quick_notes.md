# Docker Prune Commands — Quick Notes

Docker prune commands remove unused resources to reclaim disk space. Always inspect resources before cleanup, especially in production.

## 1. Common prune commands

| Command | What it removes |
|---|---|
| `docker container prune` | All stopped containers |
| `docker image prune` | Dangling images by default |
| `docker image prune -a` | Images not used by any container (including non-dangling images) |
| `docker network prune` | Networks not used by containers |
| `docker volume prune` | Volumes considered unused by Docker's applicable rules |
| `docker system prune` | Stopped containers, unused networks, dangling images, and build cache |
| `docker system prune -a` | Same system cleanup, plus unused images beyond dangling images |
| `docker system prune --volumes` | System cleanup plus eligible unused volumes |

> **Note:** Exact behavior can vary by Docker version and options. Review the confirmation prompt before approving cleanup.

## 2. Inspect before pruning

```bash
# Check disk usage by Docker
docker system df

# Show containers, including stopped ones
docker ps -a

# List images
docker images

# List networks
docker network ls

# List volumes
docker volume ls
```

## 3. Example workflow

```bash
# Remove stopped containers
docker container prune

# Remove dangling images
docker image prune

# Remove common unused resources
docker system prune
```

Docker may ask for confirmation. Add `-f` only when you intentionally want to skip the confirmation prompt.

## 4. Production best practices

- Check `docker system df` and inspect resources before cleanup.
- Confirm that stopped containers are no longer needed for debugging or recovery.
- Avoid deleting images needed for fast rollback or offline deployments.
- Be especially careful with volumes: they may contain persistent application or database data.
- Prefer targeted prune commands over broad cleanup when possible.
- In automation, understand the exact effect of flags such as `-a`, `--volumes`, and `-f` before using them.

**Remember:** Prune removes unused resources; it does not decide whether a resource is safe for your business to lose.
