# Delete Docker Images in One Command

<img width="950" height="817" alt="image" src="https://github.com/user-attachments/assets/b034fb64-f764-4059-9ae8-8262ad754545" />


## Command

```bash
docker rmi -f $(docker image ls -aq)
```

- `docker image ls -aq` — lists IDs of all local images (`-a`: all images, `-q`: IDs only).
- `docker rmi -f` — attempts to remove those images, forcing removal where Docker permits.

> **Note:** This command may fail or be disruptive if images are used by existing containers. `-f` does not bypass every dependency or guarantee that all images can be removed.

## Safer Approach

If the goal is to reclaim disk space, inspect usage first:

```bash
docker system df
docker image ls
```

Remove only selected unused images:

```bash
docker image prune
```

To remove **all unused images**, including images not referenced by any container:

```bash
docker image prune -a
```

Review the prompt before confirming. Avoid broad cleanup commands on shared or production hosts without checking impact; local image removal can affect rollback or require images to be pulled again.

**Remember:** Inspect first, remove selectively, and protect production rollback options.
