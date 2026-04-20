# Day 44 – Secrets, Artifacts & Running Real Tests in CI

## Task 1: GitHub Secrets — safe usage

### Creating the secret

Repo → Settings → Secrets and Variables → Actions → New repository secret
- Name: `MY_SECRET_MESSAGE`
- Value: (any string)

### Workflow — check secret is set without printing it

```yaml
name: Secrets Demo

on: [push]

jobs:
  secret-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Check secret is set (safe)
        env:
          MSG: ${{ secrets.MY_SECRET_MESSAGE }}
        run: |
          if [[ -n "$MSG" ]]; then
            echo "The secret is set: true"
          else
            echo "The secret is set: false"
          fi

      - name: Attempt to echo secret directly (masked by GitHub)
        run: echo "${{ secrets.MY_SECRET_MESSAGE }}"
```

When you try to echo `${{ secrets.MY_SECRET_MESSAGE }}` directly, GitHub replaces the value with `***` in the logs. The value is never exposed.

### Why you should never print secrets in CI logs

CI logs are often shared across a team, archived for months, and accessible through the GitHub UI to anyone with repo read access. Even when GitHub masks known secret values, the secret can leak through:
- Error messages that echo parts of the value
- Subprocesses that don't inherit the masking
- Base64-encoded or URL-encoded variants that bypass the mask pattern

A leaked CI token can give an attacker full access to your registry, cloud provider, or production environment.

---

## Task 2: Secrets as Environment Variables

```yaml
name: Secret Env Vars

on: [push]

jobs:
  use-secrets:
    runs-on: ubuntu-latest
    steps:
      - name: Use secret via env var
        env:
          API_TOKEN: ${{ secrets.MY_SECRET_MESSAGE }}
          DOCKER_USER: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASS: ${{ secrets.DOCKER_TOKEN }}
        run: |
          # Safe: use the env var in commands without hardcoding
          curl -s -H "Authorization: Bearer $API_TOKEN" \
               https://api.example.com/health

          echo "$DOCKER_PASS" | docker login \
               -u "$DOCKER_USER" --password-stdin
```

**Rule of thumb:** Always pass secrets via `env:` at the step level. Never interpolate `${{ secrets.X }}` directly inside a `run:` string — it writes the value into the shell script text before the runner executes it, bypassing GitHub's masking in edge cases.

**Secrets to add for Day 45:**
- `DOCKER_USERNAME` — your Docker Hub username
- `DOCKER_TOKEN` — a Docker Hub access token (Account Settings → Security → New Access Token)

---

## Task 3: Upload Artifacts

```yaml
name: Upload Artifact Demo

on: [push]

jobs:
  build-and-upload:
    runs-on: ubuntu-latest
    steps:
      - name: Generate test report
        run: |
          mkdir -p reports
          echo "Test results: $(date)" > reports/report.txt
          echo "Tests passed: 42"     >> reports/report.txt
          echo "Tests failed: 0"      >> reports/report.txt

      - name: Upload test report
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: reports/
          retention-days: 7
```

**Verify:** Actions tab → click the workflow run → scroll down to the Artifacts section → click to download the zip.

> Screenshot: [add screenshot of artifact download UI]

---

## Task 4: Download Artifacts Between Jobs

```yaml
name: Artifact Between Jobs

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Create build output
        run: |
          echo "version=1.2.3" > build-meta.txt
          echo "commit=${{ github.sha }}" >> build-meta.txt

      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: build-meta
          path: build-meta.txt

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build output
        uses: actions/download-artifact@v4
        with:
          name: build-meta

      - name: Use the artifact
        run: |
          echo "Received from build job:"
          cat build-meta.txt
```

### When would you use artifacts in a real pipeline?

Jobs run on separate VMs — they share no filesystem. Artifacts are the transport layer for anything one job produces that another job needs:

- A compiled binary built in a `build` job and consumed by a `test` job
- A test coverage report uploaded for display in a PR comment
- A Docker image layer cache passed between pipeline stages
- A versioned release bundle staged for deployment
- Log files from a flaky test that needs human inspection

---

## Task 5: Run Real Tests in CI

### Test script — `tests/test_math.py`

```python
def add(a, b):
    return a + b

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_add_negative():
    assert add(-5, -3) == -8
```

### Workflow — `.github/workflows/ci-tests.yml`

```yaml
name: CI Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install pytest

      - name: Run tests
        run: python -m pytest tests/ -v
```

**Break it:** Change `assert add(2, 3) == 5` to `assert add(2, 3) == 99` → pipeline goes red.
**Fix it:** Revert the assertion → pipeline goes green.

> Screenshot: [add screenshot of passing test run in GitHub Actions]

---

## Task 6: Dependency Caching

```yaml
name: Cached Install

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            pip-${{ runner.os }}-

      - name: Install dependencies
        run: pip install -r requirements.txt
```

### What is cached and where is it stored?

`~/.cache/pip` is pip's local download cache — the directory where pip stores downloaded wheel files and source archives before installing them. By caching this directory between runs, `pip install` can skip the network download for packages that haven't changed.

GitHub stores the cache in its own backend, keyed by `pip-<os>-<hash_of_requirements.txt>`. If `requirements.txt` hasn't changed since the last run, the cache hits and pip resolves packages locally instead of downloading from PyPI.

**Time saving:** First run (cache miss) typically takes 30–120 seconds for a medium requirements file. Second run (cache hit) usually takes 2–5 seconds. The `restore-keys:` fallback allows a partial cache hit (e.g., when a single new package is added) — pip re-downloads only the new package and reuses everything else.

**Node.js equivalent:**
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: npm-
```

---

## Secrets management — key takeaways

| Rule | Reason |
|---|---|
| Never print a secret in a `run:` step | Logs are stored and shareable |
| Pass secrets via `env:` not inline `${{ }}` in scripts | Avoids writing the value into shell script text |
| Use least-privilege tokens | A scoped Docker token beats your account password |
| Rotate secrets regularly | Limits the blast radius of a leak |
| Never store secrets in code or workflow YAML | They end up in git history forever |

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`