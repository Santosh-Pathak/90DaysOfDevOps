# Day 41 – Triggers & Matrix Builds

## Task 1: PR Trigger — `.github/workflows/pr-check.yml`

```yaml
name: PR Check

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize]

jobs:
  pr-check:
    runs-on: ubuntu-latest
    steps:
      - name: Print branch name
        run: echo "PR check running for branch: ${{ github.head_ref }}"
```

**How to test:** Create a new branch, push a commit, open a PR against `main`.
The workflow appears automatically on the PR's **Checks** tab.

---

## Task 2: Scheduled Trigger

Add this block to any existing workflow:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'   # every day at midnight UTC
```

### Cron expression — every Monday at 9 AM UTC

```
0 9 * * 1
```

Cron field order: `minute  hour  day-of-month  month  day-of-week`
- `0 9` → 9:00 AM
- `* *` → every day of the month, every month
- `1`   → Monday (0 = Sunday, 1 = Monday … 6 = Saturday)

---

## Task 3: Manual Trigger — `.github/workflows/manual.yml`

```yaml
name: Manual Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment (staging / production)'
        required: true
        default: 'staging'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Print selected environment
        run: echo "Deploying to environment: ${{ inputs.environment }}"
```

**How to trigger:** Actions tab → select *Manual Deploy* → click **Run workflow** → pick environment.

---

## Task 4: Matrix Builds — `.github/workflows/matrix.yml`

```yaml
name: Matrix Build

on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]

    steps:
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print Python version
        run: python --version
```

**Total jobs:** 3 Python versions × 2 operating systems = **6 parallel jobs**

| Job | OS | Python |
|-----|----|--------|
| 1 | ubuntu-latest | 3.10 |
| 2 | ubuntu-latest | 3.11 |
| 3 | ubuntu-latest | 3.12 |
| 4 | windows-latest | 3.10 |
| 5 | windows-latest | 3.11 |
| 6 | windows-latest | 3.12 |

---

## Task 5: Exclude & Fail-Fast

```yaml
name: Matrix with Exclude

on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]
        exclude:
          - os: windows-latest
            python-version: "3.10"

    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print version
        run: python --version
```

**Jobs after exclusion:** 6 − 1 = **5 jobs**  
The excluded combination is `Python 3.10` on `windows-latest`.

### fail-fast behaviour

| Setting | Behaviour |
|---------|-----------|
| `fail-fast: true` *(default)* | GitHub cancels all still-running and pending matrix jobs the moment any single job fails. Saves CI minutes but hides whether other combinations would also fail. |
| `fail-fast: false` | Every job runs to completion regardless of failures elsewhere. Useful for spotting which specific OS/version combinations are broken. |

---

## Key concepts summary

| Trigger | `on:` syntax | Use case |
|---------|-------------|----------|
| Push | `push:` | Run on every commit |
| Pull request | `pull_request: branches: [main]` | Validate PRs |
| Schedule | `schedule: - cron: '...'` | Nightly jobs, reports |
| Manual | `workflow_dispatch:` | On-demand deploys |
| Matrix | `strategy: matrix:` | Multi-env testing |

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`