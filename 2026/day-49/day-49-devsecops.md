# Day 49 – DevSecOps: Add Security to Your CI/CD Pipeline

---

## What DevSecOps Means (In My Own Words)

DevSecOps is the practice of embedding security checks directly into the CI/CD pipeline so that vulnerabilities, leaked secrets, and risky dependencies are caught automatically — before code ever reaches production. Instead of a separate security team reviewing things after the fact, security becomes just another automated gate in the pipeline, like a failing test.

---

## Updated Pipeline Diagram

```
PR opened
  → build & test
  → dependency vulnerability check   ← NEW (Day 49)
  → branch name check
  → PR checks PASS or FAIL

Merge to main
  → build & test
  → Docker build
  → Trivy image scan (fail on CRITICAL/HIGH)  ← NEW (Day 49)
  → Docker push (only if scan passes)
  → deploy to production (with approval gate)

Always active (GitHub platform-level)
  → Secret scanning (detect leaked API keys, tokens)  ← NEW (Day 49)
  → Push protection (block commits with secrets)      ← NEW (Day 49)
```

---

## Task 1: Trivy Docker Image Scan

Added to `main-pipeline.yml` after the Docker build step, before the push:

```yaml
# .github/workflows/main-pipeline.yml (updated)
name: Main Branch Pipeline

on:
  push:
    branches:
      - main

permissions:
  contents: read
  security-events: write   # needed to upload SARIF results

jobs:
  build-and-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.12"
      run_tests: true

  docker-build:
    runs-on: ubuntu-latest
    needs: build-and-test
    outputs:
      image_ref: ${{ steps.set-ref.outputs.image_ref }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build Docker image (local only — not pushed yet)
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          IMAGE_REF="${{ secrets.DOCKER_USERNAME }}/github-actions-capstone:sha-${SHORT_SHA}"
          docker build -t "$IMAGE_REF" .
          echo "image_ref=${IMAGE_REF}" >> $GITHUB_OUTPUT
        id: set-ref

      - name: Scan Docker image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.set-ref.outputs.image_ref }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Push Docker image (only if scan passed)
        run: |
          docker push ${{ steps.set-ref.outputs.image_ref }}
          docker tag ${{ steps.set-ref.outputs.image_ref }} \
            ${{ secrets.DOCKER_USERNAME }}/github-actions-capstone:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/github-actions-capstone:latest

  deploy:
    runs-on: ubuntu-latest
    needs: docker-build
    environment: production
    steps:
      - name: Deploy
        run: echo "🚀 Deploying ${{ needs.docker-build.outputs.image_ref }} to production"
```

### What Trivy actually checks

Trivy scans the image layers against the CVE (Common Vulnerabilities and Exposures) database maintained by NVD, GitHub Advisory, and OS-specific sources (Alpine SecDB, Debian Security Tracker, etc.). It checks:

- **OS packages** — Alpine, Debian, Ubuntu, RHEL packages installed in the image
- **Language dependencies** — Python pip packages, Node npm modules, Go modules, etc.
- **Secrets** — Accidentally embedded API keys or tokens in image layers

**`exit-code: '1'`** means the pipeline step fails (and the pipeline halts) if CRITICAL or HIGH CVEs are found. Changing to `exit-code: '0'` makes it advisory only — useful during a grace period when migrating to a new base image.

**What CVEs look like in the Trivy output:**
```
┌──────────────────┬────────────────┬──────────┬──────────────────────────────────┐
│ Library          │ Vulnerability  │ Severity │ Title                            │
├──────────────────┼────────────────┼──────────┼──────────────────────────────────┤
│ libexpat         │ CVE-2024-XXXXX │ HIGH     │ Integer overflow in XML parser    │
│ pip              │ CVE-2023-XXXXX │ CRITICAL │ Remote code execution via ...     │
└──────────────────┴────────────────┴──────────┴──────────────────────────────────┘
```

**Fix:** Switch the base image from `python:3.12` to `python:3.12-slim` or `python:3.12-alpine` — slimmer images have far fewer installed packages and fewer CVEs.

---

## Task 2: GitHub Secret Scanning

### Setup (no workflow changes needed)

1. Repo → **Settings** → **Code security and analysis**
2. Enable **Secret scanning** → On
3. Enable **Push protection** → On

### Secret scanning vs Push protection

| Feature | Secret scanning | Push protection |
|---|---|---|
| **When it runs** | After the commit is already pushed | Before the push is accepted |
| **What it does** | Scans the repo history and alerts you | Blocks the push entirely if a secret is detected |
| **Effect on existing secrets** | Sends an alert and (for some partners) revokes the token automatically | Prevents the secret from ever reaching the repo |
| **Best for** | Catching secrets already in the repo | Preventing new secrets from being pushed |

### What happens if GitHub detects a leaked AWS key

1. GitHub identifies the pattern matching AWS access key format (`AKIA...`).
2. GitHub notifies the repo owner via email and in the Security tab.
3. For AWS specifically, GitHub's partnership with AWS triggers **automatic revocation** of the key within minutes — the key becomes invalid before most attackers can use it.
4. GitHub shows a remediation guide: rotate the key, audit CloudTrail logs, check for unauthorized usage.

**With push protection enabled,** the developer gets blocked at `git push` with a message explaining which file contains the secret and offering options: bypass with justification, or remove the secret before pushing.

---

## Task 3: Dependency Review Action

Added to `pr-pipeline.yml`:

```yaml
# .github/workflows/pr-pipeline.yml (updated)
name: PR Pipeline

on:
  pull_request:
    branches:
      - main
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: critical

  build-and-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.12"
      run_tests: true

  pr-comment:
    runs-on: ubuntu-latest
    needs: [dependency-review, build-and-test]
    steps:
      - name: PR summary
        run: |
          echo "✅ PR checks passed for branch: ${{ github.head_ref }}"
          echo "Test result: ${{ needs.build-and-test.outputs.test_result }}"
```

**How it works:** The dependency review action compares the dependency manifest (`requirements.txt`, `package.json`, etc.) between the PR's base and head. Any **newly introduced** packages are checked against GitHub's Advisory Database. It only flags new additions — packages already in `main` are not re-checked on every PR.

**Why it's on PRs only:** The action requires a `pull_request` event to have a base commit to diff against. It cannot run on `push` events.

---

## Task 4: Workflow Permissions

Added `permissions` blocks to all workflow files:

### Minimal permissions (read-only workflows)

```yaml
# For workflows that only need to read code and run tests
permissions:
  contents: read
```

### PR workflows (need to read PRs and post status checks)

```yaml
permissions:
  contents: read
  pull-requests: write    # if posting PR comments
  checks: write           # if setting check statuses programmatically
```

### Security scanning workflows

```yaml
permissions:
  contents: read
  security-events: write  # required to upload SARIF results to Security tab
```

### Why limiting permissions matters

By default, GitHub Actions grants `GITHUB_TOKEN` with **write** permissions to most scopes (contents, packages, deployments, etc.). If a malicious or compromised third-party action runs in your workflow, it could:

- **Push code** to your repository (overwrite files, inject malicious code)
- **Delete branches or tags** 
- **Create releases** or **publish packages**
- **Read secrets** from the environment

Setting `permissions: contents: read` at the top of a workflow means even if a compromised action tries to `git push`, it will get a 403 — the token simply doesn't have write access.

**Principle of least privilege** — give each workflow exactly the permissions it needs, nothing more.

---

## Brownie Points: Upload Scan to Security Tab

```yaml
- name: Run Trivy (SARIF format)
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ steps.set-ref.outputs.image_ref }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload SARIF to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

SARIF (Static Analysis Results Interchange Format) is a JSON-based format that GitHub's Security tab can render with file paths, line numbers, and remediation suggestions. Uploading SARIF results means scan findings appear alongside CodeQL results in a unified security dashboard.

---

## Brownie Points: Pin Actions to Commit SHAs

Tags like `@v4` or `@master` are mutable — an action author can silently update what they point to. Pinning to a SHA guarantees you run exactly the code you audited:

```yaml
# Instead of:
uses: actions/checkout@v4

# Pin to exact commit (copy from the action's release page):
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

This is a **supply chain security** best practice. A compromised action that moves a tag could exfiltrate secrets, push malicious code, or modify your release artifacts. SHA pinning eliminates this attack vector.

---

## Full Secure Pipeline Summary

| Layer | Tool | What it catches |
|---|---|---|
| Every PR | `actions/dependency-review-action` | Newly introduced vulnerable packages |
| Every PR | Branch name check | Process / naming convention violations |
| Post Docker build | `aquasecurity/trivy-action` | CVEs in the image layers and language packages |
| Always (platform) | GitHub Secret Scanning | Leaked API keys, tokens, passwords in code |
| Always (platform) | Push Protection | Blocks secret-containing commits before they land |
| Workflow level | `permissions:` blocks | Limits blast radius if an action is compromised |