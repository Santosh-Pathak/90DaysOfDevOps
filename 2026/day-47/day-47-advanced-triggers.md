# Day 47 – Advanced Triggers: PR Events, Cron Schedules & Event-Driven Pipelines

---

## Task 1: Pull Request Lifecycle Workflow

### `pr-lifecycle.yml`

```yaml
# .github/workflows/pr-lifecycle.yml
name: PR Lifecycle Events

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  pr-info:
    runs-on: ubuntu-latest
    steps:
      - name: Print event type
        run: echo "Event fired: ${{ github.event.action }}"

      - name: Print PR details
        run: |
          echo "PR Title:         ${{ github.event.pull_request.title }}"
          echo "PR Author:        ${{ github.event.pull_request.user.login }}"
          echo "Source branch:    ${{ github.head_ref }}"
          echo "Target branch:    ${{ github.base_ref }}"

      - name: Handle merged PR
        if: github.event.action == 'closed' && github.event.pull_request.merged == true
        run: |
          echo "🎉 PR was MERGED into ${{ github.base_ref }}"
          echo "Merged by: ${{ github.event.pull_request.merged_by.login }}"
```

**How each event type fires:**

| Action | When it fires |
|---|---|
| `opened` | A new PR is created |
| `synchronize` | New commits are pushed to the PR branch |
| `reopened` | A closed (unmerged) PR is reopened |
| `closed` | PR is closed — could be merged OR just closed without merging |

The merge conditional `github.event.pull_request.merged == true` is required because both a plain close and a merge trigger the `closed` action — only merged PRs have `merged: true`.

---

## Task 2: PR Validation Workflow

### `pr-checks.yml`

```yaml
# .github/workflows/pr-checks.yml
name: PR Gate Checks

on:
  pull_request:
    branches:
      - main
    types: [opened, synchronize, reopened]

jobs:
  file-size-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # needed to diff against base branch

      - name: Check file sizes in PR
        run: |
          MAX_SIZE=1048576   # 1 MB in bytes
          FAILED=0
          CHANGED_FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)

          for FILE in $CHANGED_FILES; do
            if [ -f "$FILE" ]; then
              SIZE=$(stat -c%s "$FILE")
              if [ "$SIZE" -gt "$MAX_SIZE" ]; then
                echo "❌ $FILE is too large: ${SIZE} bytes (limit: ${MAX_SIZE} bytes)"
                FAILED=1
              else
                echo "✅ $FILE — ${SIZE} bytes"
              fi
            fi
          done

          if [ "$FAILED" -eq 1 ]; then
            echo "One or more files exceed the 1 MB limit."
            exit 1
          fi

  branch-name-check:
    runs-on: ubuntu-latest
    steps:
      - name: Validate branch name
        run: |
          BRANCH="${{ github.head_ref }}"
          echo "Branch: ${BRANCH}"

          if echo "$BRANCH" | grep -qE '^(feature|fix|docs)/.+'; then
            echo "✅ Branch name is valid: ${BRANCH}"
          else
            echo "❌ Branch name '${BRANCH}' does not follow the required pattern."
            echo "   Allowed patterns: feature/*, fix/*, docs/*"
            exit 1
          fi

  pr-body-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check PR description
        run: |
          BODY="${{ github.event.pull_request.body }}"
          if [ -z "$BODY" ]; then
            echo "⚠️  WARNING: PR description is empty. Please add a description."
            # We warn but don't fail — exit 0
          else
            echo "✅ PR has a description."
          fi
```

**Testing it:** Opening a PR from a branch named `my-random-branch` immediately fails the `branch-name-check` job. The PR cannot merge until the branch is renamed to `feature/...`, `fix/...`, or `docs/...`.

---

## Task 3: Scheduled Workflows (Cron Deep Dive)

### `scheduled-tasks.yml`

```yaml
# .github/workflows/scheduled-tasks.yml
name: Scheduled Tasks

on:
  schedule:
    - cron: '30 2 * * 1'      # Every Monday at 2:30 AM UTC
    - cron: '0 */6 * * *'     # Every 6 hours
  workflow_dispatch:            # Manual trigger for testing

jobs:
  scheduled-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print which schedule fired
        run: |
          echo "Schedule: ${{ github.event.schedule }}"
          echo "Current UTC time: $(date -u)"

      - name: Health check
        run: |
          URL="https://httpbin.org/status/200"
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$URL")

          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "✅ Health check passed — HTTP ${HTTP_CODE}"
          else
            echo "❌ Health check failed — HTTP ${HTTP_CODE}"
            exit 1
          fi
```

### Cron Expressions (Notes)

**Cron syntax: `minute hour day-of-month month day-of-week`**

| Description | Cron Expression | Explanation |
|---|---|---|
| Every weekday at 9 AM IST | `30 3 * * 1-5` | IST = UTC+5:30, so 9AM IST = 3:30 AM UTC |
| First day of every month at midnight | `0 0 1 * *` | Minute 0, Hour 0, Day 1, any month, any weekday |
| Every Monday at 2:30 AM UTC | `30 2 * * 1` | Minute 30, Hour 2, any day, any month, Monday (1) |
| Every 6 hours | `0 */6 * * *` | Minute 0, every 6th hour |

**Why GitHub may delay or skip scheduled workflows:**

GitHub's scheduler is a shared, best-effort service — not a real-time cron daemon. Two key reasons for delays or skips:

1. **Inactive repos:** If a repo has no activity for 60+ days, GitHub automatically disables scheduled workflows to conserve resources. You must re-enable them manually or trigger the workflow at least once.
2. **Heavy load periods:** During high-traffic times (top of the hour, especially 00:00 UTC), GitHub's scheduler queues many workflows simultaneously. Your run may start minutes late.

This is why `workflow_dispatch` is always added — it lets you test without waiting for the schedule, and it counts as activity to keep the scheduler alive.

---

## Task 4: Path & Branch Filters

### `smart-triggers.yml`

```yaml
# .github/workflows/smart-triggers.yml
name: Smart Path and Branch Triggers

on:
  push:
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
      - 'app/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build
        run: echo "Source or app files changed — running build"
```

### Ignore-docs variant:

```yaml
# .github/workflows/skip-docs.yml
name: Skip on Docs-Only Changes

on:
  push:
    branches:
      - main
      - 'release/**'
    paths-ignore:
      - '*.md'
      - 'docs/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: echo "Non-docs change detected — running tests"
```

### When to use `paths` vs `paths-ignore`

| Use | When |
|---|---|
| `paths` | You want the workflow to run **only** for specific directories (allowlist). Example: only build the backend when `src/` changes. |
| `paths-ignore` | You want the workflow to run for **everything except** certain files (denylist). Example: skip CI for pure documentation PRs. |

**Rule of thumb:** Use `paths` for focused pipelines (one service in a monorepo). Use `paths-ignore` for general pipelines where docs and config files shouldn't burn CI minutes.

**Testing:** Pushing a commit that only modifies `README.md` skips the `smart-triggers.yml` workflow entirely — the Actions tab shows no run started, and the commit gets a neutral status.

---

## Task 5: `workflow_run` — Chaining Workflows

### `tests.yml`

```yaml
# .github/workflows/tests.yml
name: Run Tests

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run tests
        run: |
          echo "Running test suite..."
          # Replace with: pytest, npm test, go test, etc.
          echo "All tests passed ✅"
```

### `deploy-after-tests.yml`

```yaml
# .github/workflows/deploy-after-tests.yml
name: Deploy After Tests

on:
  workflow_run:
    workflows: ["Run Tests"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Check upstream result
        run: |
          CONCLUSION="${{ github.event.workflow_run.conclusion }}"
          echo "Tests workflow conclusion: ${CONCLUSION}"

          if [ "$CONCLUSION" != "success" ]; then
            echo "⚠️ Tests did not pass (conclusion: ${CONCLUSION}). Skipping deploy."
            exit 1
          fi

          echo "✅ Tests passed — proceeding to deploy"

      - name: Deploy
        run: echo "🚀 Deploying to production..."
```

**Key behavior:** `workflow_run` fires even when the upstream workflow fails — that's why the explicit `conclusion == 'success'` check is required inside the job. Without it, deploys would run on failed test runs.

---

## Task 6: `repository_dispatch` — External Event Triggers

### `external-trigger.yml`

```yaml
# .github/workflows/external-trigger.yml
name: External Deploy Trigger

on:
  repository_dispatch:
    types: [deploy-request]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Print client payload
        run: |
          echo "Triggered by external event: deploy-request"
          echo "Target environment: ${{ github.event.client_payload.environment }}"
          echo "Requested by: ${{ github.event.client_payload.requested_by }}"

      - name: Deploy to environment
        run: |
          ENV="${{ github.event.client_payload.environment }}"
          echo "🚀 Deploying to ${ENV}..."
```

### Triggering with `gh` CLI:

```bash
gh api repos/<owner>/<repo>/dispatches \
  -f event_type=deploy-request \
  -f client_payload='{"environment":"production","requested_by":"slack-bot"}'
```

### When would an external system trigger a pipeline?

Real-world use cases:

- **Slack bot** — A `/deploy production` slash command calls the GitHub API, which fires `repository_dispatch`. The pipeline then runs the deploy.
- **Monitoring alert** — A PagerDuty/Datadog webhook detects a failed health check and automatically triggers a rollback pipeline.
- **Release management tool** — A product manager clicks "Ship to production" in an internal dashboard. The dashboard sends a `repository_dispatch` to kick off the full release workflow.
- **Cross-repo dependencies** — Repo A finishes building a shared library and dispatches an event to Repo B to rebuild using the new version.

`repository_dispatch` is the bridge between GitHub Actions and any external system that can make HTTP requests.

---

## `workflow_run` vs `workflow_call` — Explained

| | `workflow_run` | `workflow_call` |
|---|---|---|
| **How it's triggered** | Fires automatically when another named workflow completes | Called explicitly with `uses:` at the job level |
| **Coupling** | Loose — the called workflow doesn't know it will trigger the next one | Tight — the caller explicitly invokes the reusable workflow |
| **Access to upstream results** | Yes, via `github.event.workflow_run.conclusion` | Yes, via `needs.<job>.outputs` |
| **Best for** | Chaining independently-defined workflows (e.g., tests → deploy) | Sharing reusable workflow logic across multiple callers |
| **Runs in the same workflow run?** | No — creates a new, separate workflow run | Yes — shows up as a job within the caller's run |

**Simple analogy:**
- `workflow_call` = calling a function you wrote. You control both ends.
- `workflow_run` = a webhook that fires when something else finishes. You only control what happens after.

---

## Summary

| Trigger | Use Case |
|---|---|
| `pull_request` + activity types | Run different logic on PR open vs. merge vs. update |
| `schedule` (cron) | Nightly builds, health checks, weekly reports |
| `paths` / `paths-ignore` | Skip unnecessary CI runs in monorepos |
| `workflow_run` | Chain independent workflows without coupling them |
| `repository_dispatch` | Integrate external tools (Slack, monitoring, dashboards) |