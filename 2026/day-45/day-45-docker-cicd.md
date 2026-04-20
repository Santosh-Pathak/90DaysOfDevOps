# Day 45 – Docker Build & Push in GitHub Actions

## Task 1: Minimal Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

```python
# app.py
print("Hello from CI/CD!")
```

Place both files in the root of your `github-actions-practice` repo. Secrets `DOCKER_USERNAME` and `DOCKER_TOKEN` from Day 44 should already be set.

---

## Task 2 & 3: Complete Workflow — `.github/workflows/docker-publish.yml`

```yaml
name: Docker publish

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Compute short SHA
        id: vars
        run: echo "sha=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/myapp:latest
            ${{ secrets.DOCKER_USERNAME }}/myapp:sha-${{ steps.vars.outputs.sha }}
```

**Key design decisions:**
- `setup-buildx-action` enables BuildKit — faster layer caching and multi-platform builds
- The short SHA step slices the first 7 characters of `github.sha` to create a readable, traceable tag
- `push: ${{ github.ref == 'refs/heads/main' }}` evaluates to `true` on main, `false` on PRs — so the image always builds but only pushes on main

---

## Task 4: Only Push on Main

The `push:` field in `build-push-action` accepts a boolean expression:

```yaml
push: ${{ github.ref == 'refs/heads/main' }}
```

**Test it:**
1. Create branch `feature/test-no-push`, push a commit, open a PR
2. The workflow runs — build succeeds, no push happens (you can confirm on Docker Hub that the tag timestamp hasn't updated)
3. Merge the PR to main — push runs, Docker Hub shows a fresh `latest` and `sha-XXXXXXX`

---

## Task 5: Status Badge

Add to `README.md`:

```markdown
![Docker publish](https://github.com/YOUR_USERNAME/github-actions-practice/actions/workflows/docker-publish.yml/badge.svg)
```

Replace `YOUR_USERNAME` with your GitHub username. The badge reads the most recent workflow run on the default branch.

**Quick way to get the exact URL:** Actions tab → click "Docker publish" workflow → three-dot menu (top right) → "Create status badge" → copy the markdown.

---

## Task 6: Pull and Run

```bash
# Pull latest
docker pull YOUR_DOCKERHUB_USERNAME/myapp:latest

# Run it
docker run --rm YOUR_DOCKERHUB_USERNAME/myapp:latest

# Run a specific commit (traceable, reproducible)
docker run --rm YOUR_DOCKERHUB_USERNAME/myapp:sha-a3f9c12
```

**Expected output:**
```
Hello from CI/CD!
```

> Docker Hub image: https://hub.docker.com/r/YOUR_USERNAME/myapp

> Screenshot: [add screenshot of successful pipeline run in GitHub Actions]

---

## Task 6: The full journey from `git push` to a running container

```
developer            GitHub               runner VM            Docker Hub
    │                    │                     │                    │
    │── git push main ──>│                     │                    │
    │                    │── trigger workflow ─>│                    │
    │                    │                     │── checkout code    │
    │                    │                     │── setup buildx     │
    │                    │                     │── docker login ───>│
    │                    │                     │── build image      │
    │                    │                     │── push :latest ───>│
    │                    │                     │── push :sha-XXXX ─>│
    │                    │<── workflow complete─│                    │
    │                    │                     │                    │
anywhere:
    $ docker pull username/myapp:latest        │<── pull layers ────│
    $ docker run --rm username/myapp:latest    │                    │
    Hello from CI/CD!
```

1. A developer pushes to `main`
2. GitHub detects the push event and queues the workflow
3. A fresh Ubuntu VM is provisioned and the runner agent picks up the job
4. The code is checked out onto the runner's filesystem
5. Docker Buildx builds the image from the Dockerfile, layer-by-layer
6. The `docker/login-action` authenticates with Docker Hub using the stored secrets
7. Both the `:latest` and `:sha-XXXXXXX` tags are pushed to the registry
8. The runner VM is discarded — nothing persists on the runner
9. Anyone with network access can `docker pull` the exact image and run it immediately

The SHA tag is critical for real pipelines — `:latest` is a moving pointer and hard to debug. The SHA tag pins an exact, immutable build to a specific commit, enabling reproducible deployments and instant rollbacks.

---

## Useful extras

### Multi-platform build (for M1/ARM compatibility)

```yaml
- uses: docker/setup-qemu-action@v3   # needed for cross-platform

- uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```

### Layer caching (faster builds)

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    cache-from: type=gha
    cache-to: type=gha,mode=max
    tags: ...
```

`type=gha` uses GitHub Actions' built-in cache storage — Docker layers are cached between runs and only changed layers are rebuilt.

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`