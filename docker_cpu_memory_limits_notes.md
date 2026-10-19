# Docker CPU and Memory Limits — Quick Notes

## 1. Set resource limits

```bash
docker run -d \
  --name my-app \
  --cpus="1.5" \
  --memory="512m" \
  my-app:1.0.0
```

- `--cpus="1.5"`: allows the container to use up to 1.5 CPUs.
- `--memory="512m"`: sets the container memory limit to 512 MiB.

## 2. Monitor resource usage

```bash
docker stats
```

Shows live CPU, memory, network I/O, block I/O, and process usage for running containers.

## 3. Production considerations

- Choose limits using load testing, observed usage, application requirements, and host capacity.
- **Memory limit too low:** the container may run out of memory and be killed by the kernel (OOM kill).
- **CPU limit too low:** CPU throttling can increase application latency.
- Monitor usage and tune limits based on evidence; do not use identical values for every container.
