# Day 37 — Docker Revision

## Self-Assessment Checklist

- [x] Run a container from Docker Hub (interactive + detached)
- [x] List, stop, remove containers and images
- [x] Explain image layers and how caching works
- [x] Write a Dockerfile from scratch with FROM, RUN, COPY, WORKDIR, CMD
- [x] Explain CMD vs ENTRYPOINT
- [x] Build and tag a custom image
- [x] Create and use named volumes
- [x] Use bind mounts
- [x] Create custom networks and connect containers
- [x] Write a docker-compose.yml for a multi-container app
- [x] Use environment variables and .env files in Compose
- [x] Write a multi-stage Dockerfile
- [x] Push an image to Docker Hub
- [x] Use healthchecks and depends_on

> Update these marks honestly — **can do**, **shaky**, or **haven't done** — before committing.

---

## Quick-Fire Questions — Answers

**1. What is the difference between an image and a container?**

An image is a read-only blueprint — a stack of filesystem layers built from a Dockerfile. It lives on disk and never runs by itself.

A container is a live, writable instance of an image. Docker adds a thin writable layer on top of the image layers when you run it. Think of the image as a class and the container as an object instantiated from that class.

---

**2. What happens to data inside a container when you remove it?**

It is gone permanently. Containers carry a writable layer that is tied to their lifecycle — when the container is removed with `docker rm`, that layer (and everything written to it) is deleted.

To persist data beyond container death, you must use:
- **Named volumes** — Docker-managed storage outside the container lifecycle
- **Bind mounts** — a directory on the host mapped into the container

---

**3. How do two containers on the same custom network communicate?**

By container name (or service name in Compose). Docker's built-in DNS automatically resolves container names to their IP addresses within a custom network.

```
# Container "api" can reach "db" like this:
postgres://db:5432/mydb
```

This does NOT work on the default `bridge` network — you need a user-defined network for automatic DNS resolution.

---

**4. What does `docker compose down -v` do differently from `docker compose down`?**

`docker compose down` stops and removes containers and the networks Compose created. Volumes are left intact — your data survives.

`docker compose down -v` does all of the above and also removes the named volumes declared under the `volumes:` key in your `docker-compose.yml`. Use `-v` when you want a fully clean slate (e.g., resetting a database). Be careful — this is irreversible.

---

**5. Why are multi-stage builds useful?**

They keep production images small and lean. The idea: use a heavy "builder" stage (with compilers, dev dependencies, build tools) to compile/bundle your app, then copy only the compiled output into a minimal final image.

Benefits:
- Smaller attack surface (no build tools in production)
- Faster pull/deploy (smaller image = less transfer)
- No secrets or source code leaking into the final image
- One Dockerfile instead of separate build scripts

A Node.js app might go from ~900 MB to ~100 MB with a multi-stage build.

---

**6. What is the difference between `COPY` and `ADD`?**

`COPY` — simple and explicit. Copies files or directories from host into the image. The safe default choice.

`ADD` — a superset of COPY with two extra powers:
1. Auto-extracts local `.tar`, `.tar.gz`, `.tar.bz2` archives into the destination
2. Can fetch files from remote URLs (though `curl`/`wget` inside `RUN` is preferred for cache control)

Best practice: use `COPY` unless you specifically need the archive extraction. `ADD`'s extra behaviors can cause surprising results.

---

**7. What does `-p 8080:80` mean?**

It maps a port on the host to a port inside the container.

Format: `-p HOST_PORT:CONTAINER_PORT`

`-p 8080:80` means: traffic arriving at `localhost:8080` on your machine is forwarded to port `80` inside the container. The app inside only knows about port 80; the outside world reaches it via 8080.

---

**8. How do you check how much disk space Docker is using?**

```bash
docker system df
```

Output breaks down usage by:
- Images (total vs reclaimable)
- Containers (running vs stopped)
- Local volumes
- Build cache

To actually reclaim space:
```bash
docker system prune        # safe: removes stopped containers, unused networks, dangling images
docker system prune -a     # aggressive: also removes all unused images (not just dangling)
docker system prune -a -f  # skip confirmation prompt
```

---

## Weak Spots to Revisit

Pick the two topics you marked as shaky and redo the hands-on tasks:

### Suggested: Multi-stage builds

```bash
# 1. Create a simple Node app
mkdir multi-stage-demo && cd multi-stage-demo
echo '{"name":"demo","version":"1.0"}' > package.json
echo 'console.log("Hello from production!")' > server.js

# 2. Write the multi-stage Dockerfile (see cheat sheet skeleton)
# 3. Build and compare sizes
docker build -t demo:single -f Dockerfile.single .
docker build -t demo:multi  -f Dockerfile.multi  .
docker images | grep demo
```

### Suggested: Healthchecks + depends_on

```yaml
# In docker-compose.yml
db:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5

web:
  build: .
  depends_on:
    db:
      condition: service_healthy   # waits until db passes healthcheck
```

```bash
docker compose up
# Watch: web service will not start until db reports healthy
docker compose ps   # check STATUS column
```

---

## Summary

Docker's core mental model:

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Container
                                  ↓                           ↓
                              docker push              writable layer
                                  ↓                    (lost on rm)
                            Docker Hub / Registry
```

Data persistence lives outside this cycle — volumes and bind mounts are the answer.

Compose orchestrates the whole thing declaratively: services, networks, volumes, env vars, health checks — all in one file.