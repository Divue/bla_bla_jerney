# 🔄 CI/CD Pipeline Notes
### Jerney Blog Platform — DevSecOps Pipeline with GitHub Actions 

---

## Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [What is DevSecOps?](#2-what-is-devsecops)
3. [GitHub Actions — The Platform](#3-github-actions--the-platform)
4. [Pipeline Overview](#4-pipeline-overview)
5. [Pipeline File Structure](#5-pipeline-file-structure)
6. [Triggers — When Does the Pipeline Run?](#6-triggers--when-does-the-pipeline-run)
7. [Stage 1 — Lint](#7-stage-1--lint)
8. [Stage 2 — SCA (Dependency Audit)](#8-stage-2--sca-dependency-audit)
9. [Stage 3 — Build & Push Images](#9-stage-3--build--push-images)
10. [Stage 4 — Image Scan (Trivy)](#10-stage-4--image-scan-trivy)
11. [Stage 5 — IaC Scan (Checkov)](#11-stage-5--iac-scan-checkov)
12. [Stage 6 — Dockerfile Lint (Hadolint)](#12-stage-6--dockerfile-lint-hadolint)
13. [Stage 7 — Update K8s Manifest](#13-stage-7--update-k8s-manifest)
14. [Key GitHub Actions Concepts](#14-key-github-actions-concepts)
15. [The GitOps Pattern](#15-the-gitops-pattern)
16. [Interview Questions & Answers](#16-interview-questions--answers)
17. [Quick Reference Cheat Sheet](#17-quick-reference-cheat-sheet)
18. [What to Improve for Production](#18-what-to-improve-for-production)

---

## 1. What is CI/CD?

### 1.1 The Problem It Solves

Imagine 5 developers pushing code to the same app. Without automation:

- Someone pushes broken code — the app goes down at 2am
- A developer forgets to run tests before merging
- Deploying requires someone to manually SSH into a server and run commands
- A library with a known security vulnerability ships to production
- The code on production is different from what's in Git — "it works on my machine"

**CI/CD solves this by automating the entire journey from code push to deployment.**

### 1.2 What CI and CD Mean

| Term | Full Name | What it does |
|---|---|---|
| **CI** | Continuous Integration | Every code push automatically triggers: build, test, scan. Catches bugs early, before they merge. |
| **CD** | Continuous Delivery | Automatically prepares a release that is *ready* to deploy — one click to production. |
| **CD** | Continuous Deployment | Automatically deploys to production on every successful pipeline run. No human approval. |

> **Jerney uses CI + Continuous Delivery** — the pipeline builds, tests, and scans automatically, then updates the K8s manifest. The actual deployment to the cluster is done by applying the manifest (or by ArgoCD in a GitOps setup).

### 1.3 Why CI/CD Matters

```
Without CI/CD:                       With CI/CD:
──────────────                       ─────────────
Dev writes code                      Dev writes code
       ↓                                    ↓
"Works on my machine"                Pipeline runs automatically
       ↓                                    ↓
Manual testing (maybe)               Lint → Audit → Build → Scan
       ↓                                    ↓
Manual build                         If all passes → image pushed
       ↓                                    ↓
Manual deploy (fingers crossed)      Manifest updated → ready to deploy
       ↓
Production incident
```

---

## 2. What is DevSecOps?

Traditional pipeline: **Dev → Ops** (build and deploy)
DevSecOps pipeline: **Dev → Sec → Ops** (build, scan for security issues, then deploy)

The idea is **"shift left"** — catch security problems early in the development cycle (left = earlier), not after deploying to production.

```
Shift Left Security:

❌ Old way:  Code → Build → Deploy → [Security team scans] → Oops, CVE in prod
✅ New way:  Code → [Lint] → [Dependency Scan] → Build → [Image Scan] → [IaC Scan] → Deploy
                    ↑ Security checks woven into every stage
```

**Security stages in Jerney's pipeline:**

| Stage | Tool | What it catches |
|---|---|---|
| Dependency Audit | `npm audit` | Known CVEs in npm packages |
| Image Scan | Trivy | Vulnerabilities in OS packages + libraries inside the container |
| IaC Scan | Checkov | Misconfigurations in Terraform and Kubernetes manifests |
| Dockerfile Lint | Hadolint | Dockerfile bad practices that create security/performance issues |

---

## 3. GitHub Actions — The Platform

GitHub Actions is GitHub's built-in CI/CD platform. Pipelines are defined as YAML files in `.github/workflows/`. Every push triggers the relevant workflows.

### Core Concepts

| Concept | What it is |
|---|---|
| **Workflow** | The entire pipeline definition. One YAML file = one workflow. |
| **Job** | A group of steps that run on the same machine. Jobs run in parallel by default. |
| **Step** | A single task inside a job — either a shell command or an Action. |
| **Action** | A reusable unit of automation from the GitHub Marketplace (e.g. `actions/checkout@v4`). |
| **Runner** | The machine that executes jobs. `ubuntu-latest` = a fresh VM GitHub provides. |
| **Event** | What triggers the workflow (`push`, `pull_request`, `schedule`, etc.). |
| **Secret** | Encrypted values stored in GitHub settings — accessed as `${{ secrets.NAME }}`. |
| **Context** | Built-in variables like `${{ github.sha }}`, `${{ github.actor }}`. |

### File location

```
.github/
└── workflows/
    └── pipeline.yaml    ← Your pipeline lives here
```

---

## 4. Pipeline Overview

```
Push / PR to any branch
         │
         ▼
┌─────────────────────┐    ┌─────────────────────┐
│  Stage 1: Lint      │    │  Stage 6: Dockerfile │
│  (backend+frontend) │    │  Lint (hadolint)     │
└────────┬────────────┘    └─────────────────────┘
         │ needs: lint
         ▼
┌─────────────────────┐    ┌─────────────────────┐
│  Stage 2: SCA       │    │  Stage 5: IaC Scan  │
│  (npm audit)        │    │  (checkov)          │◄── needs: lint
└────────┬────────────┘    └─────────────────────┘
         │ needs: sca
         ▼
┌─────────────────────┐
│  Stage 3: Build     │
│  (docker build+push)│
└────────┬────────────┘
         │ needs: build
         ▼
┌─────────────────────┐
│  Stage 4: Image     │
│  Scan (trivy)       │
└────────┬────────────┘
         │ (only on push, not PR)
         ▼
┌─────────────────────┐
│  Stage 7: Update    │
│  K8s Manifest       │
└─────────────────────┘
```

**Key behaviours:**
- Stages 1, 5, 6 run in parallel (no dependencies between them)
- Stages 2 → 3 → 4 → 7 are sequential (each needs the previous)
- Stage 7 only runs on `push` events, not on pull requests
- Stages 1, 4, 5, 6 have `continue-on-error: true` — failures are reported but don't block the pipeline

---

## 5. Pipeline File Structure

```yaml
name: Jerney DevSecOps Pipeline   # Display name in GitHub UI

on:                               # Triggers
  push:
    branches: ["*"]
  pull_request:
    branches: ["*"]

permissions:                      # Default token permissions (read-only)
  contents: read

env:                              # Global environment variables
  DOCKER_REGISTRY: ghcr.io
  BACKEND_IMAGE: ghcr.io/${{ github.repository }}/jerney-backend
  FRONTEND_IMAGE: ghcr.io/${{ github.repository }}/jerney-frontend

jobs:                             # All jobs defined here
  lint:   ...
  sca:    ...
  build:  ...
  image-scan: ...
  iac-scan: ...
  dockerfile-lint: ...
  update-manifest: ...
```

### Global environment variables:

`env` at the top level makes variables available to ALL jobs. The image names use `${{ github.repository }}` which resolves to `owner/repo-name` — so for `iam-veeramalla/Jerney` it becomes `ghcr.io/iam-veeramalla/Jerney/jerney-backend`.

---

## 6. Triggers — When Does the Pipeline Run?

```yaml
on:
  push:
    branches: ["*"]        # Every push to every branch
  pull_request:
    branches: ["*"]        # Every PR targeting any branch
```

`["*"]` is a wildcard — matches any branch name. This means the pipeline runs on every single push and every PR open/update.

### Common trigger patterns:

```yaml
# Only run on main branch push
on:
  push:
    branches: ["main"]

# Run on push to main, PRs targeting main
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

# Only when specific files change
on:
  push:
    paths:
      - "backend/**"
      - "frontend/**"

# On a schedule (every day at midnight UTC)
on:
  schedule:
    - cron: "0 0 * * *"

# Manually triggered from GitHub UI
on:
  workflow_dispatch:
```

### Permissions:

```yaml
permissions:
  contents: read   # Default: only read the repo
```

This is a security best practice — limit what the `GITHUB_TOKEN` can do by default. Individual jobs that need more (like `build` needing to push packages) override this with their own `permissions` block.

---

## 7. Stage 1 — Lint

```yaml
lint:
  name: "🔍 Lint Code"
  runs-on: ubuntu-latest
  continue-on-error: true
  strategy:
    matrix:
      component: [backend, frontend]
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: "20"
        cache: "npm"
        cache-dependency-path: "${{ matrix.component }}/package-lock.json"

    - name: Install dependencies
      working-directory: ${{ matrix.component }}
      run: npm ci

    - name: Run ESLint
      working-directory: ${{ matrix.component }}
      run: npm run lint
```

### What is Linting?

Linting is static code analysis — checking code for errors, bugs, and style problems **without running it**. ESLint is the standard linter for JavaScript/Node.js.

Examples of what ESLint catches:
- Unused variables
- Missing semicolons
- `==` instead of `===`
- Unreachable code
- Potential null pointer issues

### Matrix Strategy:

```yaml
strategy:
  matrix:
    component: [backend, frontend]
```

This runs the job **twice in parallel** — once with `component=backend` and once with `component=frontend`. Instead of writing two separate identical jobs, the matrix handles it.

`${{ matrix.component }}` is replaced with the actual value in each run.

### `npm ci` vs `npm install`:

| `npm install` | `npm ci` |
|---|---|
| Installs packages, may update `package-lock.json` | Never modifies `package-lock.json` |
| Slower | Faster (uses lock file exactly) |
| For local development | For CI/CD — deterministic, reproducible |

### `cache: "npm"`:

Caches the npm package download cache between runs. If dependencies haven't changed (same `package-lock.json`), they're restored from cache instead of re-downloaded. Speeds up pipeline runs significantly.

### `continue-on-error: true`:

If ESLint fails, the job is marked as failed but the pipeline continues to the next stage. Used here because lint failures shouldn't block security scans.

---

## 8. Stage 2 — SCA (Dependency Audit)

```yaml
sca:
  name: "🛡️ Dependency Audit"
  runs-on: ubuntu-latest
  needs: lint
  strategy:
    matrix:
      component: [backend, frontend]
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: "20"
        cache: "npm"
        cache-dependency-path: "${{ matrix.component }}/package-lock.json"

    - run: npm ci
      working-directory: ${{ matrix.component }}

    - name: npm audit
      working-directory: ${{ matrix.component }}
      run: npm audit --audit-level=high || true
```

### What is SCA?

**Software Composition Analysis** — scanning your third-party dependencies for known security vulnerabilities. Your app uses dozens of npm packages. Each package has a history. Some have known CVEs (Common Vulnerabilities and Exposures).

```
Your app
└── express@4.18.0
    └── qs@6.11.0  ← has CVE-2022-24999 (prototype pollution)
└── lodash@4.17.20 ← has CVE-2021-23337 (command injection)
```

`npm audit` checks your `package-lock.json` against the npm advisory database and reports vulnerabilities by severity: `critical`, `high`, `moderate`, `low`.

### `--audit-level=high`:

Only fail on `high` or `critical` severity vulnerabilities. Moderate and low are reported but don't fail the job.

### `|| true`:

If `npm audit` exits with a non-zero code (found vulnerabilities), `|| true` prevents it from failing the step. The issues are still printed in the log — but they don't block the pipeline. This is a **dev-mode compromise**. In production you'd remove `|| true` so the pipeline fails hard on high/critical CVEs.

### `needs: lint`:

This job only starts after the `lint` job completes. This creates a sequential dependency — SCA waits for lint.

---

## 9. Stage 3 — Build & Push Images

```yaml
build:
  name: "🐳 Build ${{ matrix.component }}"
  runs-on: ubuntu-latest
  needs: sca
  permissions:
    contents: read
    packages: write    # Required to push to GHCR
  strategy:
    matrix:
      component: [backend, frontend]
  steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/${{ github.repository }}/jerney-${{ matrix.component }}
        tags: |
          type=sha
          type=ref,event=branch
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push image
      uses: docker/build-push-action@v6
      with:
        context: ./${{ matrix.component }}
        file: ./${{ matrix.component }}/Dockerfile
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: true
        sbom: true
```

### What is GHCR?

**GitHub Container Registry** (`ghcr.io`) — GitHub's Docker image registry. Like Docker Hub, but private to your GitHub organisation and free for public repos. Images are pushed here and pulled from here by Kubernetes.

### `docker/setup-buildx-action`:

Sets up **Docker Buildx** — an extended build client that supports:
- Multi-platform builds (build for `linux/amd64` and `linux/arm64` simultaneously)
- Advanced caching
- BuildKit features (faster, more efficient builds)

### Authentication:

```yaml
username: ${{ github.actor }}         # The user/bot that triggered the workflow
password: ${{ secrets.GITHUB_TOKEN }} # Auto-generated token — no manual secret needed
```

`GITHUB_TOKEN` is a special secret GitHub automatically creates for every workflow run. It has permissions scoped to the current repository. Here, the `packages: write` permission allows it to push images to GHCR.

### Image Tags:

```yaml
tags: |
  type=sha                                              # ghcr.io/.../jerney-backend:abc1234
  type=ref,event=branch                                 # ghcr.io/.../jerney-backend:main
  type=raw,value=latest,enable={{is_default_branch}}    # ghcr.io/.../jerney-backend:latest (only on main)
```

Each image gets 3 tags:
- **SHA tag** — exact, immutable. Used in the K8s manifest. `abc1234` identifies exactly which commit this image came from.
- **Branch tag** — `main`, `feature-login`, etc. Always points to the latest build of that branch.
- **latest** — only updated when pushing to the default branch (main). Convention for "most recent stable".

### `push: ${{ github.event_name != 'pull_request' }}`:

On pull requests: **build only, don't push**. This validates the Dockerfile builds successfully without polluting the registry with every PR image. On `push` events: build AND push.

### Build Cache:

```yaml
cache-from: type=gha    # Pull cache from GitHub Actions cache
cache-to: type=gha,mode=max    # Push all layers to GitHub Actions cache
```

Docker builds layer by layer. If a layer hasn't changed (e.g. dependencies installed), it's pulled from cache instead of rebuilt. `mode=max` caches every intermediate layer — maximum cache hits, fastest builds.

### Provenance and SBOM:

```yaml
provenance: true    # Generates a build attestation (who built it, when, from what source)
sbom: true          # Generates a Software Bill of Materials (full list of all packages inside the image)
```

- **Provenance** — a signed record proving this image was built from this specific commit by this pipeline. Prevents supply chain attacks where someone swaps your image.
- **SBOM** — a manifest of every package inside the container image. Required by some compliance frameworks (SOC2, FedRAMP). Also used by security scanners.

---

## 10. Stage 4 — Image Scan (Trivy)

```yaml
image-scan:
  name: "🔬 Scan ${{ matrix.component }} Image"
  runs-on: ubuntu-latest
  continue-on-error: true
  needs: build
  strategy:
    matrix:
      component: [backend, frontend]
  steps:
    - uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ghcr.io/${{ github.repository }}/jerney-${{ matrix.component }}:${{ github.sha }}
        format: "table"
        exit-code: "1"
        ignore-unfixed: true
        vuln-type: "os,library"
        severity: "CRITICAL,HIGH"
        timeout: "10m"
```

### What is Trivy?

**Trivy** (by Aqua Security) is a comprehensive vulnerability scanner for:
- Container images (OS packages, application libraries)
- Filesystems
- Git repositories
- Kubernetes manifests
- Terraform configs

It scans the image that was just pushed and reports CVEs found inside it.

### Key options explained:

| Option | Value | Meaning |
|---|---|---|
| `image-ref` | `...jerney-backend:${{ github.sha }}` | Scan the image tagged with the exact commit SHA — the one just built |
| `format` | `table` | Print results as a human-readable table in the logs |
| `exit-code` | `1` | Exit with code 1 (failure) if vulnerabilities found — would fail the pipeline if not for `continue-on-error` |
| `ignore-unfixed` | `true` | Skip vulnerabilities with no available fix — no point failing on things you can't fix yet |
| `vuln-type` | `os,library` | Scan both OS-level packages (apt, apk) and application libraries (npm, pip) |
| `severity` | `CRITICAL,HIGH` | Only report critical and high severity — ignore medium/low noise |
| `timeout` | `10m` | Abort if scan takes more than 10 minutes |

### SCA vs Image Scan — what's the difference?

| SCA (Stage 2) | Image Scan (Stage 4) |
|---|---|
| Scans your `package.json` dependencies | Scans everything INSIDE the built container image |
| Only sees your direct + transitive npm packages | Also sees the base OS, system packages (glibc, openssl, etc.) |
| Catches: vulnerable npm packages | Catches: vulnerable OS packages + npm packages + system libraries |
| Runs before build | Runs after build |

They complement each other — SCA is faster and earlier, image scan is more complete.

---

## 11. Stage 5 — IaC Scan (Checkov)

```yaml
iac-scan:
  name: "🏗️ IaC Security Scan"
  runs-on: ubuntu-latest
  continue-on-error: true
  needs: lint
  steps:
    - uses: actions/checkout@v4

    - name: Run Checkov on Terraform
      uses: bridgecrewio/checkov-action@v12
      with:
        directory: terraform/
        framework: terraform
        download_external_modules: true
        soft_fail: false
        skip_check: CKV_AWS_39,CKV_AWS_58
        output_format: cli

    - name: Run Checkov on Kubernetes manifests
      uses: bridgecrewio/checkov-action@v12
      with:
        directory: k8s/
        framework: kubernetes
        soft_fail: true
        output_format: cli
```

### What is Checkov?

**Checkov** scans Infrastructure as Code files (Terraform, K8s YAML, Dockerfiles, CloudFormation) for security misconfigurations — before they're deployed.

Examples of what Checkov catches:

**In Terraform:**
- EKS public endpoint enabled (no IP restriction)
- S3 bucket without versioning
- RDS without encryption
- Security group open to 0.0.0.0/0

**In Kubernetes:**
- Container running as root
- No resource limits set
- `readOnlyRootFilesystem` not set
- No liveness/readiness probes
- Secrets stored in env vars instead of secretRef

### `skip_check: CKV_AWS_39,CKV_AWS_58`:

Some Checkov rules are intentionally violated for valid reasons:
- `CKV_AWS_39` — EKS public endpoint check. We need it public in dev so we can `kubectl` from our laptop.
- `CKV_AWS_58` — another public endpoint related check.

Skipping specific checks acknowledges the risk intentionally rather than ignoring it silently.

### `soft_fail` difference:

```yaml
# Terraform scan:
soft_fail: false    # Failures would cause the job to fail (but continue-on-error saves us)

# Kubernetes scan:
soft_fail: true     # Always exits 0 — purely informational, never fails the job
```

K8s scan is informational only because some Checkov K8s rules are overly strict for learning projects.

### `download_external_modules: true`:

Downloads the Terraform modules referenced in `main.tf` (`terraform-aws-modules/vpc`, `terraform-aws-modules/eks`) so Checkov can scan through them for issues too, not just your top-level files.

---

## 12. Stage 6 — Dockerfile Lint (Hadolint)

```yaml
dockerfile-lint:
  name: "📋 Dockerfile Lint"
  runs-on: ubuntu-latest
  continue-on-error: true
  strategy:
    matrix:
      component: [backend, frontend]
  steps:
    - uses: actions/checkout@v4

    - name: Run Hadolint
      uses: hadolint/hadolint-action@v3.1.0
      with:
        dockerfile: ${{ matrix.component }}/Dockerfile
        failure-threshold: warning
```

### What is Hadolint?

**Hadolint** (Haskell Dockerfile Linter) checks Dockerfiles against best practices. It understands shell commands inside `RUN` instructions and applies shellcheck rules too.

Examples of what Hadolint catches:

| Issue | Rule | Why it matters |
|---|---|---|
| `apt-get install` without `apt-get update` | DL3009 | Installs stale packages from outdated cache |
| No version pinned: `apt-get install curl` | DL3008 | Non-reproducible — different versions each build |
| `ADD` used instead of `COPY` | DL3020 | `ADD` has implicit tar extraction — security risk |
| `sudo` used | DL3004 | Don't use sudo in containers |
| `latest` tag used: `FROM node:latest` | DL3007 | Non-reproducible, unpredictable |
| `COPY` instead of `.dockerignore` to exclude | DL3010 | Copies unnecessary files |

### `failure-threshold: warning`:

Fail the step if any issue of severity `warning` or higher is found. The severity levels are: `error > warning > info > style`.

---

## 13. Stage 7 — Update K8s Manifest

```yaml
update-manifest:
  name: "🚀 Update K8s Manifest"
  runs-on: ubuntu-latest
  if: github.event_name == 'push'    # Only on push, not PRs
  permissions:
    contents: write                  # Need to push a git commit
  steps:
    - uses: actions/checkout@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}

    - name: Configure git
      run: |
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"

    - name: Sync repository
      run: git pull --rebase origin ${{ github.ref_name }}

    - name: Get short SHA
      id: sha
      run: echo "short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Update K8s manifest with new image tags
      run: |
        IMAGE_TAG=${{ steps.sha.outputs.short }}

        BACKEND_LINE=$(awk '/- name: backend/{found=1} found && /image:/{print NR; exit}' k8s/jerney.yaml)
        sed -i "${BACKEND_LINE}s|.*|          image: ${BACKEND_IMAGE}:${IMAGE_TAG}|" k8s/jerney.yaml

        FRONTEND_LINE=$(awk '/- name: frontend/{found=1} found && /image:/{print NR; exit}' k8s/jerney.yaml)
        sed -i "${FRONTEND_LINE}s|.*|          image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|" k8s/jerney.yaml

    - name: Commit and push manifest update
      run: |
        git add k8s/jerney.yaml
        git diff --cached --quiet || git commit -m "ci: update k8s images to ${{ steps.sha.outputs.short }} [skip ci]"
        git push origin ${{ github.ref_name }}
```

### What this stage does

This is the **GitOps update step**. After the new image is built and pushed to GHCR, the K8s deployment manifest needs to reference the new image tag. This step:

1. Gets the short SHA of the current commit (e.g. `abc1234`)
2. Finds the exact line numbers where `image:` appears after `name: backend` and `name: frontend`
3. Replaces those lines with the new image tag
4. Commits and pushes the change back to the repo

The result: `k8s/jerney.yaml` in Git always reflects exactly which image version is "desired".

### Breaking down the `awk` + `sed` commands:

```bash
# awk: find the line number of the image field for a specific container
BACKEND_LINE=$(awk '/- name: backend/{found=1} found && /image:/{print NR; exit}' k8s/jerney.yaml)
# /- name: backend/  → when this pattern is found, set found=1
# found && /image:/  → when found=1 AND this line contains "image:", print the line number (NR) and exit
# Result: BACKEND_LINE = "42" (the line number)

# sed: replace the entire content of that line with the new image
sed -i "${BACKEND_LINE}s|.*|          image: ${BACKEND_IMAGE}:${IMAGE_TAG}|" k8s/jerney.yaml
# -i         → edit file in place
# 42s        → substitute on line 42 only
# |.*|       → match anything (the whole line)
# |          image: ghcr.io/.../jerney-backend:abc1234|  → replace with this
```

### Git identity configuration:

```bash
git config user.name "github-actions[bot]"
git config user.email "github-actions[bot]@users.noreply.github.com"
```

Git requires a name and email for commits. Using `github-actions[bot]` makes it clear this commit was automated, not made by a human.

### `git diff --cached --quiet ||`:

```bash
git diff --cached --quiet || git commit -m "..."
```

`git diff --cached --quiet` exits with code 0 if there are NO staged changes, code 1 if there ARE changes.
- If no changes (image tag didn't change): skip the commit
- If changes exist: make the commit

This prevents empty commits (which would happen if the pipeline runs twice on the same SHA).

### `[skip ci]` in commit message:

```bash
git commit -m "ci: update k8s images to abc1234 [skip ci]"
```

`[skip ci]` tells GitHub Actions not to trigger a new pipeline run from this commit. Without it, the bot's commit would trigger the pipeline again, which would make another commit, which triggers the pipeline again — an infinite loop.

### `git pull --rebase`:

```bash
git pull --rebase origin ${{ github.ref_name }}
```

If multiple jobs run simultaneously and both try to push, one will fail because the remote has moved forward. Pulling with rebase syncs the local branch first. This prevents push conflicts.

---

## 14. Key GitHub Actions Concepts

### `needs` — Job Dependencies

```yaml
sca:
  needs: lint        # sca starts only after lint completes successfully

build:
  needs: sca         # build starts only after sca completes

# Multiple dependencies:
deploy:
  needs: [build, image-scan, iac-scan]   # Wait for ALL of these
```

Without `needs`, all jobs run in parallel simultaneously.

### `if` — Conditional Execution

```yaml
# Only run on push events (not PRs)
if: github.event_name == 'push'

# Only run on main branch
if: github.ref == 'refs/heads/main'

# Only run if previous job succeeded
if: success()

# Always run (even if previous failed) — useful for cleanup
if: always()
```

### Step outputs — passing data between steps

```yaml
- name: Get short SHA
  id: sha                                          # Give this step an ID
  run: echo "short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT  # Write output

- name: Use the value
  run: echo "Tag is ${{ steps.sha.outputs.short }}"  # Reference it
```

`$GITHUB_OUTPUT` is a special file. Writing `key=value` to it creates an output that other steps can read via `${{ steps.step-id.outputs.key }}`.

### Contexts — Built-in Variables

| Context | Examples |
|---|---|
| `github.*` | `github.sha`, `github.actor`, `github.repository`, `github.ref_name`, `github.event_name` |
| `secrets.*` | `secrets.GITHUB_TOKEN`, `secrets.ANY_SECRET` |
| `env.*` | `env.BACKEND_IMAGE` |
| `steps.*` | `steps.sha.outputs.short` |
| `matrix.*` | `matrix.component` |

### `permissions` — Security Scoping

```yaml
# Top-level default (read-only for all jobs)
permissions:
  contents: read

# Job-level override (only this job can write packages)
jobs:
  build:
    permissions:
      contents: read
      packages: write
```

Least-privilege: only grant permissions a job actually needs. If the pipeline is compromised, a read-only token can't push malicious images.

---

## 15. The GitOps Pattern

### What is GitOps?

GitOps is a deployment strategy where **Git is the single source of truth** for both application code AND infrastructure state. The rule is simple:

> If it's not in Git, it doesn't exist.

```
Traditional deployment:
  Developer → kubectl apply → Cluster
  (cluster state not always in sync with what's in Git)

GitOps deployment:
  Developer → Push to Git → Pipeline updates manifest → ArgoCD watches Git → ArgoCD applies to Cluster
  (cluster always mirrors what's in Git)
```

### How Jerney implements GitOps:

```
1. Developer pushes code to GitHub
2. Pipeline builds new Docker image → pushed to GHCR with SHA tag
3. Pipeline updates k8s/jerney.yaml with new image tag
4. k8s/jerney.yaml is committed back to Git
5. (Optional) ArgoCD detects the change in Git → applies to cluster automatically
```

Stage 7 is the **GitOps bridge** — it keeps the manifest in Git in sync with the latest built image.

### Why update the manifest in Git vs apply directly?

| Direct `kubectl apply` in pipeline | GitOps manifest update |
|---|---|
| Cluster state diverges from Git | Git always matches cluster |
| Hard to audit who deployed what | Full Git history = full deploy audit trail |
| Rollback = figure out old command | Rollback = `git revert` the manifest commit |
| No approval gates possible | PR review before merge = deployment approval |

---

## 16. Core Questions

**Q: What is CI/CD and why is it used?**
> CI (Continuous Integration) automatically builds and tests code on every push — catching bugs before they merge. CD (Continuous Delivery/Deployment) automates the release process. Together they eliminate manual, error-prone deployments, ensure every release is tested, and let teams ship faster with more confidence. In Jerney, every push triggers a 7-stage pipeline covering lint, security scans, build, and manifest update.

**Q: What is DevSecOps? What is "shift left"?**
> DevSecOps integrates security into every stage of the CI/CD pipeline instead of bolting it on at the end. "Shift left" means moving security checks earlier (left on the timeline) — catching vulnerabilities in code and dependencies before they reach production. Jerney's pipeline has 4 security stages: SCA (npm audit), image scanning (Trivy), IaC scanning (Checkov), and Dockerfile linting (Hadolint).

**Q: What is the difference between SCA and image scanning?**
> SCA (Software Composition Analysis) scans your declared dependencies (`package.json`) for known CVEs before building. Image scanning (Trivy) scans the actual built container image — including the base OS, system packages, and all libraries baked in. SCA is faster and earlier; image scanning is more complete since it sees everything inside the container, not just what your `package.json` declares.

**Q: What is Trivy? What does it scan?**
> Trivy is an open-source vulnerability scanner by Aqua Security. It scans container images for CVEs in OS packages (apk, apt) and application libraries (npm, pip, gem). In Jerney it scans the freshly pushed image tagged with the commit SHA, reporting only CRITICAL and HIGH severity issues with available fixes. It can also scan filesystems, Git repos, Terraform, and K8s manifests.

**Q: What is Checkov? What does it catch?**
> Checkov is a static analysis tool for Infrastructure as Code. It checks Terraform, Kubernetes YAML, CloudFormation, and Dockerfiles against hundreds of security and compliance rules. In Jerney it scans both `terraform/` and `k8s/` directories. Examples: detecting EKS nodes without encryption, K8s containers running as root, missing resource limits, or S3 buckets without versioning.

**Q: Why does Stage 7 (manifest update) only run on push, not pull_request?**
> On pull requests, the code hasn't been merged and reviewed yet — it shouldn't update production manifests. We also skip the `docker push` on PRs (build validates the Dockerfile works but doesn't pollute the registry). Stage 7 only makes sense after code is merged/pushed to a real branch because that's when the image is actually pushed to GHCR and the manifest needs to reference it.

**Q: What is `[skip ci]` in a commit message?**
> It's a convention that tells GitHub Actions not to trigger a new pipeline run for that commit. Stage 7 makes an automated git commit to update the K8s manifest. Without `[skip ci]`, that commit would trigger the pipeline again, which would build a new image, make another commit, trigger the pipeline again — an infinite loop. `[skip ci]` breaks the cycle.

**Q: What is GitOps? How does Jerney implement it?**
> GitOps is a pattern where Git is the single source of truth for both code and infrastructure state. Cluster state should always mirror what's in Git. Jerney implements it by having Stage 7 update `k8s/jerney.yaml` with the new image tag after every build and commit it back to Git. This means Git always reflects exactly what should be running. A tool like ArgoCD can then watch Git and automatically apply changes to the cluster — making deployment fully automated and auditable.

**Q: What is a matrix strategy in GitHub Actions?**
> A matrix strategy runs a job multiple times in parallel with different parameter values. In Jerney, `matrix: component: [backend, frontend]` runs the lint, build, and scan jobs twice simultaneously — once for the backend and once for the frontend. Instead of writing duplicate jobs, the matrix handles both with `${{ matrix.component }}` as the variable. Dramatically reduces YAML duplication.

**Q: What is `GITHUB_TOKEN` and why is it used?**
> `GITHUB_TOKEN` is a short-lived token GitHub automatically generates for each workflow run. It has permissions scoped to the current repository and expires when the job ends. It's used for pushing images to GHCR (with `packages: write`) and for committing the manifest update (with `contents: write`). It's more secure than storing a personal access token as a secret because it's automatically rotated.

---

## 17. Quick Reference Cheat Sheet

### Pipeline Stages Summary

| Stage | Job name | Tool | Runs on | `continue-on-error` |
|---|---|---|---|---|
| 1 — Lint | `lint` | ESLint | push + PR | ✅ yes |
| 2 — Dependency Audit | `sca` | npm audit | push + PR | ❌ no |
| 3 — Build & Push | `build` | Docker Buildx | push + PR | ❌ no |
| 4 — Image Scan | `image-scan` | Trivy | push + PR | ✅ yes |
| 5 — IaC Scan | `iac-scan` | Checkov | push + PR | ✅ yes |
| 6 — Dockerfile Lint | `dockerfile-lint` | Hadolint | push + PR | ✅ yes |
| 7 — Manifest Update | `update-manifest` | git + sed + awk | push only | ❌ no |

### Job Dependency Chain

```
lint ──────────────┬──── sca ──── build ──── image-scan
                   │                               │
                   └──── iac-scan     update-manifest (push only)
dockerfile-lint (independent)
```

### Security Tools Reference

| Tool | Type | What it scans | Severity levels |
|---|---|---|---|
| ESLint | Linter | JS/TS code quality | error, warn, info |
| npm audit | SCA | npm dependency CVEs | critical, high, moderate, low |
| Trivy | Image scanner | Container CVEs | CRITICAL, HIGH, MEDIUM, LOW |
| Checkov | IaC scanner | Terraform + K8s misconfigs | CRITICAL, HIGH, MEDIUM, LOW |
| Hadolint | Dockerfile linter | Dockerfile best practices | error, warning, info, style |

### Common GitHub Actions Contexts

```yaml
${{ github.sha }}           # Full commit SHA (40 chars): abc1234...
${{ github.sha }}           # Short SHA via: git rev-parse --short HEAD
${{ github.ref_name }}      # Branch name: main, feature-login
${{ github.actor }}         # Username of who triggered the run
${{ github.repository }}    # owner/repo-name: iam-veeramalla/Jerney
${{ github.event_name }}    # push, pull_request, schedule, workflow_dispatch
${{ secrets.GITHUB_TOKEN }} # Auto-generated repo-scoped token
${{ matrix.component }}     # Current matrix value: backend or frontend
${{ steps.sha.outputs.short }} # Output from a previous step
```

---

## 18. What to Improve for Production

### Remove `|| true` from npm audit
```yaml
run: npm audit --audit-level=high
# Without || true, the pipeline FAILS on high/critical CVEs.
# This is what you want in production — vulnerabilities block deployment.
```

### Add ArgoCD for Automatic Deployment
Currently Stage 7 updates the manifest in Git but someone still needs to run `kubectl apply`. ArgoCD watches the Git repo and automatically applies changes to the cluster when the manifest changes — completing the GitOps loop.

### Add `needs: [build, image-scan, iac-scan]` to manifest update
```yaml
update-manifest:
  needs: [build, image-scan, iac-scan]   # Don't deploy if scans found issues
```
Currently `update-manifest` only needs the `build` job. Adding the scan jobs as dependencies means the manifest is only updated if all security scans pass.

### Add Branch Protection Rules
In GitHub Settings → Branches → Protection Rules:
- Require the pipeline to pass before merging a PR
- Require at least 1 reviewer approval
- Block force pushes to main

This means broken or insecure code can't bypass the pipeline and merge directly.

### Separate pipeline for Terraform
Currently IaC scanning is part of the app pipeline. A separate workflow for `terraform plan` on PRs and `terraform apply` on merge to main would be cleaner — following the same CI/CD pattern but for infrastructure changes.

### Pin Action versions to commit SHAs
```yaml
# Vulnerable: a tag can be moved to point to malicious code
uses: actions/checkout@v4

# Secure: pinned to an immutable commit SHA
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```
Supply chain attacks on GitHub Actions are real. Pinning to SHAs means even if an action's tag is hijacked, your pipeline uses the exact version you audited.

---

*End of Notes — Every push is now a fully audited, scanned, automated deployment. Ship with confidence! 🚀*