# Day 43 – Jobs, Steps, Env Vars & Conditionals

## Task 1: Multi-Job Workflow — `.github/workflows/multi-job.yml`

```yaml
name: Multi-Job Pipeline

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building the app"

  test:
    needs: build          # waits for build to succeed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running tests"

  deploy:
    needs: test           # waits for test to succeed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
```

**Dependency chain:** `build → test → deploy`

The `needs:` keyword declares a dependency. If `build` fails, `test` is skipped, and so is `deploy`. GitHub renders this as a visual graph in the Actions tab.

---

## Task 2: Environment Variables

```yaml
name: Env Var Demo

on: [push]

env:
  APP_NAME: myapp           # workflow level — available everywhere

jobs:
  env-demo:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: staging  # job level — available in this job's steps

    steps:
      - name: Print all env vars
        env:
          VERSION: "1.0.0"  # step level — only this step
        run: |
          echo "App:         $APP_NAME"
          echo "Environment: $ENVIRONMENT"
          echo "Version:     $VERSION"
          echo "Commit SHA:  ${{ github.sha }}"
          echo "Triggered by: ${{ github.actor }}"
```

**Scope rules:** workflow → job → step. A step can read vars from any enclosing scope. A job-level var is not visible outside that job.

---

## Task 3: Job Outputs

```yaml
name: Job Outputs Demo

on: [push]

jobs:
  set-date:
    runs-on: ubuntu-latest
    outputs:
      today: ${{ steps.d.outputs.date }}   # expose this step's output
    steps:
      - id: d
        run: echo "date=$(date +%F)" >> $GITHUB_OUTPUT

  use-date:
    needs: set-date
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build date is ${{ needs.set-date.outputs.today }}"
```

### What `needs:` and `outputs:` do

`needs:` tells GitHub to delay a job until the listed jobs finish. It also gives the dependent job access to the outputs of those jobs via the `needs.<job-id>.outputs.<name>` context.

`outputs:` on a job declares which step outputs to publish upward. A step writes to `$GITHUB_OUTPUT` using `key=value` syntax; the job's `outputs:` block maps those step outputs to job-level names that other jobs can read.

### Why pass outputs between jobs?

Jobs run in completely isolated virtual machines — they share no memory or filesystem. Outputs are the only way to pass a computed value (a version string, a generated artifact path, a test count, a build ID) from one job to a downstream job without uploading it as an artifact.

---

## Task 4: Conditionals

```yaml
name: Conditionals Demo

on: [push, pull_request]

jobs:
  conditional-demo:
    # job-level: only runs on push, skipped on pull_request
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      # only when on the main branch
      - name: Main-only step
        if: github.ref == 'refs/heads/main'
        run: echo "This is a main branch push"

      # step that is allowed to fail
      - name: Risky step
        id: risky
        continue-on-error: true
        run: exit 1

      # runs when any earlier step failed
      - name: On-failure handler
        if: failure()
        run: echo "A previous step failed — sending alert"
```

### `continue-on-error: true`

Allows the step to exit with a non-zero code without failing the job. The job continues to the next step and the overall job result remains `success` (unless a different step fails without `continue-on-error`). Use it for optional steps like linters, coverage reports, or notifications where failure should not block the pipeline.

### Common `if:` expressions

| Expression | Meaning |
|---|---|
| `github.ref == 'refs/heads/main'` | Running on the main branch |
| `github.event_name == 'push'` | Triggered by a push |
| `failure()` | Any earlier step failed |
| `success()` | All earlier steps succeeded (default) |
| `always()` | Run regardless of previous step results |
| `cancelled()` | The workflow was cancelled |

---

## Task 5: Smart Pipeline — `.github/workflows/smart-pipeline.yml`

```yaml
name: Smart Pipeline

on:
  push:
    branches: '**'     # any branch

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Linting..."

  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."

  summary:
    needs: [lint, test]    # waits for both; lint and test run in parallel
    runs-on: ubuntu-latest
    steps:
      - name: Print branch type
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "Main branch push — production candidate"
          else
            echo "Feature branch push: ${{ github.ref_name }}"
          fi

      - name: Print commit message
        run: echo "Commit: ${{ github.event.commits[0].message }}"
```

**Job graph:**
```
lint  ─┐
        ├─→ summary
test  ─┘
```

`lint` and `test` have no `needs:` so they start in parallel immediately. `summary` depends on both, so it waits until both complete before running.

---

## Key concepts summary

| Concept | Syntax | Purpose |
|---|---|---|
| Job dependency | `needs: job-name` | Run a job only after another succeeds |
| Workflow env var | Top-level `env:` block | Available in all jobs and steps |
| Job env var | `env:` under a job | Available in all steps of that job |
| Step env var | `env:` under a step | Available only in that one step |
| Step output | `echo "key=val" >> $GITHUB_OUTPUT` | Write a value for other jobs to consume |
| Job output | `outputs: name: ${{ steps.id.outputs.key }}` | Expose a step output to dependent jobs |
| Read output | `${{ needs.job-id.outputs.name }}` | Read an upstream job's output |
| Step conditional | `if: github.ref == '...'` | Skip or run a step based on context |
| Failure handler | `if: failure()` | Run only when a previous step failed |
| Soft failure | `continue-on-error: true` | Allow a step to fail without failing the job |

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`