# Day 38 – YAML Basics

## Overview

YAML (YAML Ain't Markup Language) is the foundation of every CI/CD pipeline — from GitHub Actions to Kubernetes manifests. This document covers what I learned writing YAML by hand and validating it.

---

## YAML Files Created

### `person.yaml`

```yaml
name: Alex DevOps
role: DevOps Engineer
experience_years: 2
learning: true

tools:
  - docker
  - kubernetes
  - terraform
  - ansible
  - jenkins

hobbies: [reading, hiking, open-source contributing]
```

### `server.yaml`

```yaml
server:
  name: prod-web-01
  ip: 192.168.1.10
  port: 8080

database:
  host: db.internal
  name: appdb
  credentials:
    user: dbadmin
    password: s3cur3p@ss

startup_script: |
  #!/bin/bash
  echo "Starting services..."
  systemctl start nginx
  systemctl start app

startup_script_folded: >
  This is a long description of the startup process
  that will be folded into a single line when parsed,
  making it easier to read in the YAML file.
```

---

## Task Notes

### Task 2: Two Ways to Write a List in YAML

**Block style** — one item per line with a dash:
```yaml
tools:
  - docker
  - kubernetes
```

**Inline/flow style** — comma-separated in square brackets:
```yaml
hobbies: [reading, hiking, coding]
```

Use block style for readability when the list is long. Use inline when the list is short and you want to keep it compact.

---

### Task 3: Tab vs Spaces

When a tab is inserted instead of spaces, YAML validators throw an error like:

```
found character '\t' that cannot start any token
```

YAML strictly forbids tabs for indentation. Two spaces per level is the standard. This trips up almost every beginner because most editors insert tabs by default — configure your editor to use spaces!

---

### Task 4: `|` vs `>` — When to Use Each

| Style | Symbol | Behavior | Use When |
|-------|--------|----------|----------|
| Literal Block | `\|` | Preserves all newlines exactly | Shell scripts, code blocks, anything where line breaks matter |
| Folded Block | `>` | Collapses newlines into spaces | Long descriptions, comments, prose text where line breaks don't matter |

**Example:**
```yaml
# | keeps every line break — perfect for scripts
startup_script: |
  echo "line 1"
  echo "line 2"

# > folds into one line — perfect for descriptions
description: >
  This long text will be
  joined into a single line.
```

---

### Task 5: Validation

Validated both files using `yamllint`:

```bash
pip install yamllint
yamllint person.yaml
yamllint server.yaml
```

**Intentionally broken indentation error:**

```yaml
database:
  host: db.internal
 name: appdb   # ← wrong indentation (1 space instead of 2)
```

Error produced:
```
wrong indentation: expected 2 but found 1
```

After fixing indentation back to consistent 2-space alignment, both files passed with no errors.

---

### Task 6: Spot the Difference

**Block 1 — Correct:**
```yaml
name: devops
tools:
  - docker
  - kubernetes
```

**Block 2 — Broken:**
```yaml
name: devops
tools:
- docker
  - kubernetes
```

**What's wrong:** The list items are inconsistently indented. `- docker` is at the root level (0 spaces), but `- kubernetes` is indented 2 spaces. YAML expects all items in the same list to be at the same indentation level. Either both should be indented under `tools:`, or neither should be — but they can't be mixed.

**Fixed version:**
```yaml
name: devops
tools:
  - docker
  - kubernetes
```

---

## 3 Key Learnings

1. **Spaces only, never tabs.** This is non-negotiable in YAML. Configure your editor (VS Code, Vim, etc.) to insert spaces when you press Tab. The error `found character '\t' that cannot start any token` means you have a tab hiding somewhere.

2. **Indentation IS the structure.** YAML has no closing brackets or braces — indentation defines nesting. Two spaces per level is the community standard. One wrong space can make a deeply nested key appear at the wrong level with no obvious error.

3. **`true`/`false` are not strings.** `learning: true` is a boolean. `learning: "true"` is a string. This matters in pipelines — a condition checking for a boolean `true` will fail silently if the value is actually the string `"true"`. Quote values only when you mean it.

---

## Submission

Files added to `2026/day-38/`:
- `person.yaml`
- `server.yaml`
- `day-38-yaml.md`

---

name: Alex DevOps
role: DevOps Engineer
experience_years: 2
learning: true

tools:
  - docker
  - kubernetes
  - terraform
  - ansible
  - jenkins

hobbies: [reading, hiking, open-source contributing]


server:
  name: prod-web-01
  ip: 192.168.1.10
  port: 8080

database:
  host: db.internal
  name: appdb
  credentials:
    user: dbadmin
    password: s3cur3p@ss

startup_script: |
  #!/bin/bash
  echo "Starting services..."
  systemctl start nginx
  systemctl start app

startup_script_folded: >
  This is a long description of the startup process
  that will be folded into a single line when parsed,
  making it easier to read in the YAML file.

  
*Day 38 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*