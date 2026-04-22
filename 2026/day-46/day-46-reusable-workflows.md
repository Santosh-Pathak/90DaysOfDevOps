# Day 46 – Reusable Workflows & Composite Actions

## Task 1: Understanding `workflow_call`

### 1. What is a Reusable Workflow?
A **reusable workflow** is a GitHub Actions workflow file that can be called (invoked) by other workflows — either in the same repo or across different repos. Instead of copy-pasting the same job logic, you define it once and call it like a function. This promotes DRY (Don't Repeat Yourself) principles in CI/CD pipelines.

### 2. What is the `workflow_call` trigger?
`workflow_call` is an event trigger that marks a workflow as **callable** by other workflows. Unlike `push` or `pull_request` (which respond to repository events), `workflow_call` responds only when another workflow explicitly calls it using `uses:` at the job level. It also allows you to define `inputs`, `outputs`, and `secrets` — making it behave like a function signature.

### 3. How is calling a reusable workflow different from using a regular action (`uses:`)?
| Aspect | Regular Action (`uses:` in a step) | Reusable Workflow (`uses:` in a job) |
|---|---|---|
| Scope | Used inside a **step** | Used as an entire **job** |
| Definition | `action.yml` with `runs:` | A full workflow `.yml` with `on: workflow_call` |
| Can contain multiple jobs? | No | Yes |
| Can run on different runners? | No (inherits the job's runner) | Yes (each job defines its own runner) |
| Output passing | Via `outputs:` in `action.yml` | Via `on: workflow_call: outputs:` |

### 4. Where must a reusable workflow file live?
A reusable workflow **must** live in `.github/workflows/` — the same directory as regular workflows. For cross-repository usage, the file must be in a public repo (or a repo the caller has access to), and you reference it as `org/repo/.github/workflows/file.yml@branch`.

---

## Task 2: Reusable Workflow — `reusable-build.yml`

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      app_name:
        description: "Name of the application to build"
        type: string
        required: true
      environment:
        description: "Target deployment environment"
        type: string
        required: true
        default: staging
    secrets:
      docker_token:
        required: true
    outputs:
      build_version:
        description: "The generated build version string"
        value: ${{ jobs.build.outputs.build_version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      build_version: ${{ steps.set-version.outputs.build_version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print build info
        run: |
          echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"
          echo "Docker token is set: true"

      - name: Generate build version
        id: set-version
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          VERSION="v1.0-${SHORT_SHA}"
          echo "build_version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Generated build version: ${VERSION}"
```

**Key Points:**
- The `on: workflow_call` block defines this as a reusable workflow.
- `inputs` and `secrets` act as the function signature — callers must supply `app_name` and `docker_token`.
- The `outputs` block at the `workflow_call` level exposes `build_version` to callers, referencing `jobs.build.outputs.build_version`.
- Secrets are **never** printed — only a confirmation that they are set.

---

## Task 3: Caller Workflow — `call-build.yml`

```yaml
# .github/workflows/call-build.yml
name: Call Reusable Build Workflow

on:
  push:
    branches:
      - main

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      app_name: "my-web-app"
      environment: "production"
    secrets:
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  print-version:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Print build version from reusable workflow
        run: |
          echo "Build version received: ${{ needs.build.outputs.build_version }}"
```

**Key Points:**
- The `uses:` keyword is at the **job level** (not step level) — this is what makes it a reusable workflow call.
- `with:` passes inputs; `secrets:` passes secrets.
- `print-version` job uses `needs: build` to wait for the reusable workflow and access its outputs via `needs.build.outputs.build_version`.

---

## Task 4: Outputs Flow Explanation

The output chain works like this:

```
Step (set-version) 
  → sets: $GITHUB_OUTPUT "build_version=v1.0-abc1234"
  
Job (build) 
  → exposes: outputs.build_version = steps.set-version.outputs.build_version

Reusable Workflow (on: workflow_call) 
  → exposes: outputs.build_version = jobs.build.outputs.build_version

Caller Workflow (call-build.yml)
  → reads: needs.build.outputs.build_version
```

---

## Task 5: Composite Action — `setup-and-greet/action.yml`

```yaml
# .github/actions/setup-and-greet/action.yml
name: Setup and Greet
description: "Prints a greeting in the specified language, shows system info, and sets an output"

inputs:
  name:
    description: "Name of the person to greet"
    required: true
  language:
    description: "Language for the greeting (en, es, fr, hi)"
    required: false
    default: "en"

outputs:
  greeted:
    description: "Whether a greeting was printed"
    value: ${{ steps.greet.outputs.greeted }}

runs:
  using: "composite"
  steps:
    - name: Print greeting
      id: greet
      shell: bash
      run: |
        NAME="${{ inputs.name }}"
        LANG="${{ inputs.language }}"

        case "$LANG" in
          es) echo "¡Hola, ${NAME}! Bienvenido." ;;
          fr) echo "Bonjour, ${NAME}! Bienvenue." ;;
          hi) echo "नमस्ते, ${NAME}! स्वागत है।" ;;
          *)  echo "Hello, ${NAME}! Welcome." ;;
        esac

        echo "greeted=true" >> $GITHUB_OUTPUT

    - name: Print system info
      shell: bash
      run: |
        echo "Current date: $(date)"
        echo "Runner OS: ${{ runner.os }}"
```

### Workflow that uses the composite action:

```yaml
# .github/workflows/use-greet-action.yml
name: Use Setup and Greet Action

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run custom greeting action (English)
        id: english-greet
        uses: ./.github/actions/setup-and-greet
        with:
          name: "DevOps Engineer"
          language: "en"

      - name: Run custom greeting action (Hindi)
        id: hindi-greet
        uses: ./.github/actions/setup-and-greet
        with:
          name: "DevOps Engineer"
          language: "hi"

      - name: Check greeted output
        run: |
          echo "Was greeted (en): ${{ steps.english-greet.outputs.greeted }}"
          echo "Was greeted (hi): ${{ steps.hindi-greet.outputs.greeted }}"
```

**Key Points:**
- The composite action lives at `.github/actions/setup-and-greet/action.yml`.
- `runs: using: "composite"` is what distinguishes it from a Docker or JavaScript action.
- Each step in a composite action must specify `shell:` explicitly.
- It's called with `uses: ./.github/actions/setup-and-greet` at the **step level** (not job level).

---

## Task 6: Reusable Workflow vs Composite Action

| | Reusable Workflow | Composite Action |
|---|---|---|
| Triggered by | `on: workflow_call` in the called file | `uses:` in a **step** of a job |
| Can contain jobs? | ✅ Yes — can have multiple jobs | ❌ No — only steps |
| Can contain multiple steps? | ✅ Yes (within each job) | ✅ Yes |
| Lives where? | `.github/workflows/` directory | Any directory (e.g., `.github/actions/<name>/action.yml`) or a separate repo |
| Can accept secrets directly? | ✅ Yes — via `on: workflow_call: secrets:` | ❌ No — secrets must be passed as inputs (not recommended) or the calling job must handle them |
| Best for | Full CI/CD pipelines shared across teams/repos (build, test, deploy flows) | Reusable sequences of shell steps within a job (setup, tooling, utilities) |

### Summary of When to Use Which

**Use a Reusable Workflow when:**
- You need to share an entire pipeline (multiple jobs) across repos
- Different jobs need different runners
- You want to enforce secrets handling at the workflow level
- Example: a shared `deploy-to-k8s.yml` called by every microservice repo

**Use a Composite Action when:**
- You have a set of steps you repeat inside many jobs
- You want a lightweight, step-level reuse without spinning up separate jobs
- Example: a `setup-node-and-cache/action.yml` that installs Node, sets up caches, and runs linting

---

## Workflow Run Screenshot (Description)

When the caller workflow (`call-build.yml`) is pushed to `main`, the GitHub Actions tab shows:

```
✅ Call Reusable Build Workflow
   └── build (Reusable Build Workflow / build)
   │    ├── Checkout code
   │    ├── Print build info
   │    │    └── "Building my-web-app for production"
   │    │    └── "Docker token is set: true"
   │    └── Generate build version
   │         └── "Generated build version: v1.0-a3f9d12"
   └── print-version
        └── Print build version from reusable workflow
             └── "Build version received: v1.0-a3f9d12"
```

The caller workflow's job links directly into the reusable workflow's jobs — GitHub displays them as nested under the same workflow run. Inputs are visible in the job logs, but the secret value is never exposed.

---

## Key Takeaways

1. **`workflow_call`** makes a workflow callable like a function — define inputs, secrets, and outputs.
2. **Caller syntax**: `uses: ./.github/workflows/file.yml` (same repo) or `uses: org/repo/.github/workflows/file.yml@main` (cross-repo).
3. **Composite actions** use `runs: using: "composite"` and live in `action.yml` — called at the step level.
4. **Outputs flow**: step → job → workflow → caller, each level must explicitly forward the value.
5. **Secrets**: reusable workflows can accept secrets via `on: workflow_call: secrets:` — composite actions cannot.
6. A single workflow run can call at most **20 unique reusable workflows**.