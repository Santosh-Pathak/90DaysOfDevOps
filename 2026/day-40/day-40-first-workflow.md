# Day 40 – Your First GitHub Actions Workflow

## Setup

1. Created a new public GitHub repository: `github-actions-practice`
2. Cloned it locally: `git clone https://github.com/<your-username>/github-actions-practice.git`
3. Created the folder structure:
   ```
   mkdir -p .github/workflows
   ```

---

## Task 2: The Workflow File

`.github/workflows/hello.yml`:

```yaml
name: Hello GitHub Actions

on:
  push:
    branches:
      - "**"

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the code
        uses: actions/checkout@v4

      - name: Say hello
        run: echo "Hello from GitHub Actions!"

      - name: Print current date and time
        run: date

      - name: Print the branch that triggered this run
        run: echo "Branch is ${{ github.ref_name }}"

      - name: List files in the repo
        run: ls -la

      - name: Print the runner operating system
        run: echo "Runner OS is $RUNNER_OS"
```

Pushed with:
```bash
git add .github/workflows/hello.yml
git commit -m "day 40: first GitHub Actions workflow"
git push origin main
```

Then navigated to the **Actions** tab on GitHub and watched the run complete. ✅

---

## Task 3: Anatomy — What Each Key Does

| Key | What it does |
|-----|-------------|
| `name:` (workflow level) | Human-readable name for the whole workflow. Shows up in the GitHub Actions tab. |
| `on:` | Defines the **trigger** — the event that causes GitHub to run this workflow. Here: any `push` to any branch. Other options: `pull_request`, `schedule`, `workflow_dispatch` (manual). |
| `jobs:` | Contains all the jobs in this workflow. Each key under `jobs:` is a job ID (e.g., `greet`). |
| `runs-on:` | Tells GitHub which **runner** (virtual machine) to use for the job. `ubuntu-latest` is the most common. Options: `windows-latest`, `macos-latest`, or a self-hosted runner. |
| `steps:` | An ordered list of actions to run inside a job. Steps run **sequentially** — if one fails, the job stops. |
| `uses:` | Runs a **pre-built action** from the GitHub Actions Marketplace. `actions/checkout@v4` is the official action to clone your repo onto the runner. The `@v4` pins the version. |
| `run:` | Executes a **shell command** directly on the runner. On `ubuntu-latest` this is bash. |
| `name:` (step level) | Optional label for a step. Shows up in the job log in the Actions tab — makes it easy to find which step printed what. |

---

## Task 4: Extended Workflow Steps

The updated workflow adds four new steps:

**Print current date and time:**
```yaml
- name: Print current date and time
  run: date
```
Output: `Sun Apr 19 14:32:17 UTC 2026`

**Print branch name using a GitHub context variable:**
```yaml
- name: Print the branch that triggered this run
  run: echo "Branch is ${{ github.ref_name }}"
```
`${{ github.ref_name }}` is a **GitHub context expression**. GitHub injects dozens of these automatically — commit SHA, actor name, repo name, event type. They use the `${{ ... }}` syntax and are evaluated before the shell sees the command.

Output: `Branch is main`

**List files in the repo:**
```yaml
- name: List files in the repo
  run: ls -la
```
Shows every file the `actions/checkout@v4` step pulled onto the runner. Useful for confirming your files are actually there before later steps try to use them.

**Print the runner OS:**
```yaml
- name: Print the runner operating system
  run: echo "Runner OS is $RUNNER_OS"
```
`$RUNNER_OS` is a **default environment variable** GitHub sets automatically on every runner. On `ubuntu-latest` it prints `Linux`.

Output: `Runner OS is Linux`

---

## Task 5: Breaking It on Purpose

Added a failing step:
```yaml
- name: This step will fail
  run: exit 1
```

**What a failed pipeline looks like:**

- The Actions tab shows a red ✗ next to the commit message and workflow name.
- Inside the job, every step that ran before the failure shows green ✓.
- The failing step shows a red ✗ with the exit code (`Process completed with exit code 1.`).
- Every step listed after the failing step shows as **skipped** — GitHub stops executing once a step fails.
- GitHub sends an email notification: "Run failed: Hello GitHub Actions".

**How to read the error:**

1. Click the red ✗ run in the Actions tab.
2. Click the job name (`greet`).
3. Click the failing step — it expands to show the raw log output.
4. The last line tells you the exit code. For a real command failure (e.g., a misspelled command), you see the shell error: `bash: misspelledcmd: command not found` followed by `exit code 127`.

**Fixed by removing the failing step and pushing again** — the next run went green. ✅

---

## Key Learnings

1. **Workflow files live in `.github/workflows/`** and must end in `.yml` (or `.yaml`). GitHub scans this folder automatically on every event — no registration or setup required.

2. **`uses:` vs `run:` is the core distinction.** `uses:` runs a packaged, reusable action (written by someone else, versioned). `run:` runs raw shell. Most real pipelines mix both — `uses: actions/checkout@v4` to get your code, then `run: npm test` to test it.

3. **GitHub context variables (`${{ github.* }}`) give you metadata for free.** Branch name, actor, commit SHA, event type — all available without any configuration. Essential for conditional logic later (e.g., "only deploy if the branch is `main`").

---

## Submission

Files added to `2026/day-40/`:
- `.github/workflows/hello.yml`
- `day-40-first-workflow.md`

name: Hello GitHub Actions

on:
  push:
    branches:
      - "**"

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the code
        uses: actions/checkout@v4

      - name: Say hello
        run: echo "Hello from GitHub Actions!"

      - name: Print current date and time
        run: date

      - name: Print the branch that triggered this run
        run: echo "Branch is ${{ github.ref_name }}"

      - name: List files in the repo
        run: ls -la

      - name: Print the runner operating system
        run: echo "Runner OS is $RUNNER_OS"