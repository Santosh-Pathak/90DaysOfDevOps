# Day 48 – GitHub Actions Project: End-to-End CI/CD Pipeline

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         PR OPENED / UPDATED                      │
│                                                                   │
│   PR → [reusable-build-test.yml]                                 │
│           ├── Checkout code                                       │
│           ├── Set up Python/Node                                  │
│           ├── Install dependencies                                │
│           └── Run tests → output: test_result                    │
│                                                                   │
│         → [pr-comment job]                                        │
│           └── "PR checks passed for branch: feature/xyz"         │
│                                                                   │
│   ❌ No Docker build on PRs                                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         PUSH TO MAIN                             │
│                                                                   │
│   Job 1 → [reusable-build-test.yml]                              │
│             └── tests → test_result: passed                       │
│                                                                   │
│   Job 2 → [reusable-docker.yml]  (needs: Job 1)                  │
│             ├── Docker build                                      │
│             ├── Push tag: latest                                  │
│             ├── Push tag: sha-<short-sha>                        │
│             └── output: image_url                                 │
│                                                                   │
│   Job 3 → [deploy]  (needs: Job 2)                               │
│             ├── environment: production                           │
│             ├── (manual approval gate)                            │
│             └── "Deploying image_url to production"              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      EVERY 12 HOURS                              │
│                                                                   │
│   [health-check.yml]                                             │
│     ├── Pull latest Docker image                                  │
│     ├── Run container (detached)                                  │
│     ├── Wait 5s → curl /health                                   │
│     ├── Print PASSED / FAILED                                     │
│     ├── Stop & remove container                                   │
│     └── Write $GITHUB_STEP_SUMMARY report                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Task 1: Project App

A minimal Python Flask app (`app.py`) serves as the base:

```python
# app.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({"message": "Hello from github-actions-capstone!"})

@app.route("/health")
def health():
    return jsonify({"status": "ok"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .

EXPOSE 5000
CMD ["python", "app.py"]
```

```text
# requirements.txt
flask==3.0.3
```

```bash
# test.sh — basic integration test
#!/bin/bash
set -e
echo "Running health check..."
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)
if [ "$RESPONSE" -eq 200 ]; then
  echo "✅ Health check passed (HTTP 200)"
else
  echo "❌ Health check failed (HTTP $RESPONSE)"
  exit 1
fi
```

---

## Task 2: Reusable Workflow — Build & Test

```yaml
# .github/workflows/reusable-build-test.yml
name: Reusable Build and Test

on:
  workflow_call:
    inputs:
      python_version:
        type: string
        required: false
        default: "3.12"
      run_tests:
        type: boolean
        required: false
        default: true
    outputs:
      test_result:
        description: "passed or failed"
        value: ${{ jobs.build-test.outputs.test_result }}

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:
      test_result: ${{ steps.run-tests.outputs.test_result }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python_version }}

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        id: run-tests
        if: inputs.run_tests == true
        run: |
          # Start app in background, run test script
          python app.py &
          APP_PID=$!
          sleep 3   # let Flask start

          if bash test.sh; then
            echo "test_result=passed" >> $GITHUB_OUTPUT
          else
            echo "test_result=failed" >> $GITHUB_OUTPUT
            kill $APP_PID
            exit 1
          fi

          kill $APP_PID

      - name: Skip tests (run_tests=false)
        if: inputs.run_tests == false
        run: |
          echo "Tests skipped as requested."
          echo "test_result=skipped" >> $GITHUB_OUTPUT
```

---

## Task 3: Reusable Workflow — Docker Build & Push

```yaml
# .github/workflows/reusable-docker.yml
name: Reusable Docker Build and Push

on:
  workflow_call:
    inputs:
      image_name:
        type: string
        required: true
      tag:
        type: string
        required: true
    secrets:
      docker_username:
        required: true
      docker_token:
        required: true
    outputs:
      image_url:
        description: "Full Docker image URL with tag"
        value: ${{ jobs.docker.outputs.image_url }}

jobs:
  docker:
    runs-on: ubuntu-latest
    outputs:
      image_url: ${{ steps.set-image-url.outputs.image_url }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.docker_username }}
          password: ${{ secrets.docker_token }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.docker_username }}/${{ inputs.image_name }}:${{ inputs.tag }}

      - name: Set image URL output
        id: set-image-url
        run: |
          IMAGE_URL="${{ secrets.docker_username }}/${{ inputs.image_name }}:${{ inputs.tag }}"
          echo "image_url=${IMAGE_URL}" >> $GITHUB_OUTPUT
          echo "Image pushed: ${IMAGE_URL}"
```

---

## Task 4: PR Pipeline

```yaml
# .github/workflows/pr-pipeline.yml
name: PR Pipeline

on:
  pull_request:
    branches:
      - main
    types: [opened, synchronize]

jobs:
  build-and-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.12"
      run_tests: true

  pr-comment:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - name: PR summary
        run: |
          echo "PR checks passed for branch: ${{ github.head_ref }}"
          echo "Test result: ${{ needs.build-and-test.outputs.test_result }}"
```

---

## Task 5: Main Branch Pipeline

```yaml
# .github/workflows/main-pipeline.yml
name: Main Branch Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.12"
      run_tests: true

  docker-build-push:
    needs: build-and-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: "github-actions-capstone"
      tag: "sha-${{ github.sha }}"
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  docker-tag-latest:
    needs: build-and-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: "github-actions-capstone"
      tag: "latest"
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-tag-latest]
    environment: production   # Requires approval if protection rules are set
    steps:
      - name: Deploy to production
        run: |
          IMAGE_URL="${{ needs.docker-build-push.outputs.image_url }}"
          echo "🚀 Deploying image: ${IMAGE_URL} to production"
          # In reality: kubectl set image, helm upgrade, etc.
```

**Note on the `sha` tag:** The full SHA is 40 characters. A more readable short SHA can be generated as `$(echo ${{ github.sha }} | cut -c1-7)` and stored in an env var or a step output before passing it to the reusable workflow.

---

## Task 6: Scheduled Health Check

```yaml
# .github/workflows/health-check.yml
name: Scheduled Health Check

on:
  schedule:
    - cron: '0 */12 * * *'   # Every 12 hours
  workflow_dispatch:

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Pull latest Docker image
        run: |
          docker pull ${{ secrets.DOCKER_USERNAME }}/github-actions-capstone:latest

      - name: Run container
        run: |
          docker run -d \
            --name healthcheck-container \
            -p 5000:5000 \
            ${{ secrets.DOCKER_USERNAME }}/github-actions-capstone:latest

      - name: Wait for startup
        run: sleep 5

      - name: Run health check
        id: check
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)
          echo "http_code=${HTTP_CODE}" >> $GITHUB_OUTPUT

          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "STATUS=PASSED" >> $GITHUB_ENV
            echo "✅ Health check PASSED — HTTP ${HTTP_CODE}"
          else
            echo "STATUS=FAILED" >> $GITHUB_ENV
            echo "❌ Health check FAILED — HTTP ${HTTP_CODE}"
          fi

      - name: Stop and remove container
        if: always()   # Run even if health check step fails
        run: |
          docker stop healthcheck-container
          docker rm healthcheck-container

      - name: Write step summary
        if: always()
        run: |
          echo "## Health Check Report" >> $GITHUB_STEP_SUMMARY
          echo "| Field | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|---|---|" >> $GITHUB_STEP_SUMMARY
          echo "| Image | \`github-actions-capstone:latest\` |" >> $GITHUB_STEP_SUMMARY
          echo "| Status | **${{ env.STATUS }}** |" >> $GITHUB_STEP_SUMMARY
          echo "| HTTP Code | ${{ steps.check.outputs.http_code }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Time (UTC) | $(date -u) |" >> $GITHUB_STEP_SUMMARY
          echo "| Runner OS | ${{ runner.os }} |" >> $GITHUB_STEP_SUMMARY

      - name: Fail job if health check failed
        if: env.STATUS == 'FAILED'
        run: exit 1
```

The `if: always()` on the container cleanup step is important — it ensures the container is removed even when the health check step errors, preventing orphaned containers from accumulating on the runner.

---

## Task 7: README Badges & What's Next

### Status badges for `README.md`

```markdown
![PR Pipeline](https://github.com/<owner>/github-actions-capstone/actions/workflows/pr-pipeline.yml/badge.svg)
![Main Pipeline](https://github.com/<owner>/github-actions-capstone/actions/workflows/main-pipeline.yml/badge.svg)
![Health Check](https://github.com/<owner>/github-actions-capstone/actions/workflows/health-check.yml/badge.svg)
```

### What I'd add next

1. **Slack notifications** — Post to a `#deployments` channel when a deploy succeeds or fails using the `slackapi/slack-github-action`.
2. **Multi-environment promotion** — Add `staging` and `production` environments. The pipeline promotes `staging` automatically on every merge to `main`, and `production` requires a manual approval gate.
3. **Rollback workflow** — A `repository_dispatch`-triggered workflow that takes a previous image tag and re-deploys it. Runnable from a Slack slash command.
4. **Trivy image scanning** — Scan the Docker image after build; fail the pipeline on CRITICAL CVEs. (Done in Day 49!)
5. **OIDC authentication** — Replace long-lived Docker Hub credentials with OIDC tokens for AWS ECR / GCP Artifact Registry, eliminating stored secrets entirely.

---

## Docker Hub Image

Image pushed to: `docker.io/<your-username>/github-actions-capstone`

Tags produced per main-branch push:
- `:latest` — always the most recent build
- `:sha-<7-char-sha>` — immutable, traceable to a specific commit