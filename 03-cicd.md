# CI/CD Pipeline Standards

These standards define the structure, behavior, and requirements for all
CI/CD pipelines across projects. Specific tooling (scanners, linters, test
runners) is defined in `04-quality-gates.md`. This document defines the
pipeline shape, branch strategy, and behavioral rules that all pipelines
must follow.

All pipelines are implemented in **GitHub Actions** unless the project overlay
specifies otherwise with a documented reason.

---

## Core Principles

- **The pipeline is the authority.** If it passes, the change is shippable. If
  it fails, the change is not — regardless of how confident the author is.
- **Pipelines are code.** Workflow files are version controlled, reviewed in PRs,
  and held to the same standards as application code.
- **No manual steps in the critical path.** Any step that requires a human to
  run a command, click a button, or SSH into a server is a process gap. Automate
  it or document it as a tracked gap.
- **Fast feedback first.** Order pipeline stages so the fastest, cheapest checks
  run first. Do not make a developer wait 10 minutes for a scan to fail on
  something a linter could catch in 30 seconds.
- **Pipelines are reproducible.** The same commit must produce the same result
  on every run. Flaky pipelines are treated as bugs and fixed before new
  features are added.

---

## Branch Strategy

All projects use a release branch model. Work is developed on short-lived
feature branches, integrated into `develop`, and promoted to `main` via a
`release/*` branch and PR.

```
main                         ← protected; production-ready; tagged releases only
develop                      ← protected; integration branch; always passing CI
├── feature/<ticket-id>-short-description
├── fix/<ticket-id>-short-description
└── chore/<ticket-id>-short-description
release/<version>            ← cut from develop; PR targets main for official release
```

### Rules

- **`main` is always production-ready.** Every commit on `main` has passed the
  full pipeline including DAST. Merges to `main` come only from `release/*`
  branches via PR — never directly from feature branches.
- **`develop` is the integration target.** Feature branches PR into `develop`,
  not `main`. `develop` runs the full CI pipeline (Stages 1–3) on every push.
- **`release/*` branches are the promotion path.** When `develop` is stable and
  ready for release, a `release/<version>` branch is cut from `develop`. The PR
  from `release/<version>` to `main` triggers the full pipeline including
  integration tests and ZAP full scan before merge.
- **`develop` also publishes images.** The build and publish pipeline runs on
  push to both `develop` and `main`, producing tagged images (`develop` tag from
  `develop`, semantic version tags from `main`). Images from `develop` are
  considered pre-release and not promoted to `latest`.
- **Feature branches are short-lived.** Branches older than 5 days without
  activity should be reviewed and either progressed or closed.
- **Branch names follow the convention above.** Agents must use the pattern
  `<type>/<ticket-id>-<short-description>`. No freeform names.
- **No direct commits to `main`.** All changes reach `main` via a `release/*`
  PR. Branch protection rules enforce this.
- **One concern per branch.** A branch that fixes a bug and adds a feature is
  two branches. Keep scope tight.

### Team Workflow Profiles

Declare the active profile in the project root `AGENTS.md`.

#### Enterprise (default)

- Feature branches open PRs into `develop`.
- No direct commits to `main` or `develop`.
- Release promotion via `release/*` PR to `main`.

#### Solo / small-team

- Direct merge or push to `develop` is allowed.
- **PR still required** for `release/*` → `main`.
- No direct commits to `main`.

---

## CI Trigger Profiles

Declare which profile the project uses in the root `AGENTS.md`. Workflow file
names are overlay-defined (e.g. `publish.yml`, `docker-publish.yml`).

#### Release-gate (default)

- `pr-checks.yml` (or equivalent): PR to `main` — Stages 1–2 + ZAP full scan
- `publish.yml`: push to `main` or `develop` — Stage 1 + Stage 3 + ZAP baseline
- `integration.yml` (optional): PR to `main` — Stage 4

#### Integration-gate

- `pr-checks.yml`: PR to `develop` — Stage 1 (+ optional Stage 2 subset)
- Full Stages 1–2 + ZAP full on `release/*` PR to `main`
- `publish.yml`: push to `main` or `develop`

---

## Pipeline Stages

Every project pipeline must implement these stages. Stages within a group run
in parallel where possible, but stage groups execute in order. Typical workflow
files map to these stages (names are overlay-defined):

```
.github/
└── workflows/
    ├── pr-checks.yml          ← Stages 1–2 + ZAP full (trigger: see CI profile)
    ├── publish.yml            ← Stages 1 + 3 + ZAP baseline (push main/develop)
    └── integration.yml        ← Stage 4 (optional)
```

### Stage 1 — Fast Checks (parallel, must pass before Stage 2 and Stage 3)
Runs on: every PR and every push to `main` and `develop`.

- [ ] **Lint**: Code style and formatting validation (Ruff, ESLint)
- [ ] **Format check**: Formatting validation (Ruff format, Prettier)
- [ ] **Type check**: Static type validation (mypy, tsc --noEmit)
- [ ] **Secret scan**: Detect committed secrets in the diff (Gitleaks)
- [ ] **Unit tests + coverage**: Fast tests, no external dependencies (pytest)

### Stage 2 — Security Scanning (parallel, must pass before merge)
Runs per the project's CI trigger profile (see above). At minimum on every
release PR to `main`. Projects should also run SCA and secret scanning on push
to `develop` as an early warning layer when using the integration-gate profile.

- [ ] **SAST**: Static application security testing (Bandit, ESLint security plugin)
- [ ] **SCA — Python**: Dependency vulnerability scan (pip-audit)
- [ ] **SCA — Node**: Dependency vulnerability scan (npm audit)
- [ ] **OSS license scan**: Dependency license compatibility check
- [ ] **IaC scan**: Dockerfile and workflow misconfiguration scan (Trivy config mode)

### Stage 3 — Build and Container Scan
Runs on: push to `main` and `develop` (publish pipeline). Triggered only after
Stage 1 passes.

- [ ] **Build container image**: Docker build (single-arch `:scan` tag for scanning)
- [ ] **Container scan**: Trivy image scan — blocks push on HIGH/CRITICAL
- [ ] **ZAP baseline scan**: Passive DAST scan against containerized app
- [ ] **Build and push**: Multi-arch image push to registry — only runs if all
      above pass (`if: success()`)

### Stage 3 (PR variant) — ZAP Full Scan
Runs on: release PR to `main` (or per CI trigger profile). Complements Stage 2.

- [ ] **Build image for scan**: Single-arch build, not pushed
- [ ] **ZAP full scan**: Active DAST scan against containerized app — blocks
      merge on HIGH findings

### Stage 4 — Integration (optional)
Runs when the project defines integration tests. Trigger and workflow file are
overlay-defined. Skip this workflow entirely if the project has no integration
tests.

- [ ] **Integration tests**: Tests requiring external dependencies
- [ ] **Contract tests**: API contract validation (if applicable)

### Promotion to `main`

A `release/<version>` PR to `main` must pass:
- All of Stage 1
- All of Stage 2
- ZAP full scan (Stage 3 PR variant)
- Stage 4 integration tests (if defined)

The publish pipeline then runs on merge to `main`, executing Stage 3 (build,
Trivy scan, ZAP baseline, multi-arch push) before tagging and publishing the
release image.

---

## Pipeline Behavior Rules

### On Failure
- A failing stage stops all downstream stages immediately.
- Failure notifications are sent to the channel or method defined in the
  project overlay.
- Agents must not re-trigger a failed pipeline without addressing the root
  cause. Retrying a failing pipeline without a code or config change is not
  a fix.

### On Flakiness
- A test or scan that fails intermittently without a code change is a flaky
  test. It must be quarantined (excluded from the blocking gate) and tracked
  as a bug immediately — not silently retried.
- Flaky tests older than 14 days without resolution are escalated.

### Timeouts
- Every job must define an explicit timeout. No job runs indefinitely.
- Recommended defaults (project overlay may adjust):

  | Stage | Timeout |
  |-------|---------|
  | Lint / type check | 5 minutes |
  | Unit tests | 10 minutes |
  | Security scans | 15 minutes |
  | Build | 20 minutes |
  | Integration tests | 20 minutes |
  | DAST | 30 minutes |

### Caching
- Dependency caches are used for all package managers to reduce pipeline
  duration. Cache keys must include the lockfile hash so stale caches are
  invalidated on dependency changes.
- Build caches (Docker layer cache, compiled artifacts) are used where the
  pipeline platform supports them.
- Caches are never used to skip security scans. Scans always run against
  the current state.

---

## GitHub Actions Specific Standards

### Workflow File Structure

```
.github/
├── workflows/
│   ├── pr-checks.yml          ← lint, type, secrets, tests, SAST, SCA, ZAP full
│   ├── publish.yml            ← push main/develop: build, Trivy, ZAP baseline, push
│   └── integration.yml        ← optional: integration and contract tests
└── scripts/
    ├── zap-full-scan.sh
    └── zap-scan.sh
```

Workflow file names are **overlay-defined** (`publish.yml`, `docker-publish.yml`,
etc.). `integration.yml` is optional — omit it when the project has no
integration tests.

Workflow files are split by trigger and concern. A single monolithic workflow
is not acceptable for projects beyond a trivial prototype — failure attribution
becomes unclear and job parallelism is harder to reason about.

Each workflow file must define:
- An explicit `permissions` block (default `contents: read`)
- A `timeout-minutes` on every job
- All Actions pinned to a full commit SHA (see `04-quality-gates.md`)

### Security Hardening

- **Pin all Actions to a full commit SHA**, not a tag.
  - ✅ `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`
  - ❌ `uses: actions/checkout@v4`
  - Tags are mutable and can be hijacked. SHAs are immutable.
- **Minimal permissions per job.** Set `permissions` explicitly at the job
  level. Default to `read-only` and add only what is required.
  ```yaml
  permissions:
    contents: read
    pull-requests: write
  ```
- **Never use `pull_request_target` with untrusted code.** This trigger runs
  with write permissions and access to secrets — it is a common supply chain
  attack vector.
- **Secrets are never echoed.** Do not `echo` or `print` secrets in any step,
  even for debugging. GitHub masks known secrets but not derived values.
- **Third-party Actions are reviewed before use.** Prefer official Actions
  (GitHub, the tool vendor) over community Actions. Audit the source before
  pinning.

### Workflow Authoring for Agents

The `dev-standards` submodule contains **documentation only** — no workflow
files. When a project needs CI, agents author workflow files in the **project's**
`.github/workflows/` using the patterns below.

#### When to create each workflow

| Workflow | Create when | Typical triggers |
|----------|-------------|------------------|
| `pr-checks.yml` | Always (containerized projects) | Per CI trigger profile |
| `publish.yml` | Project builds container images | Push to `main`, `develop` |
| `integration.yml` | Project has integration tests | PR/push per overlay |
| Maintenance workflows | Operational need (registry cleanup, scheduled rebuild) | Schedule / manual |

Not every project needs every workflow. Declare which apply in the root
`AGENTS.md`.

#### Required elements (every workflow file)

- Explicit top-level or per-job `permissions` (default `contents: read`)
- `timeout-minutes` on every job
- All Actions pinned to full commit SHAs (see `04-quality-gates.md`)
- Job groupings mapped to pipeline stages in this document

#### Example: Stage 1 job group (embed in project workflow)

```yaml
permissions:
  contents: read

jobs:
  fast-checks:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      # actions/checkout v4.2.2
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - name: Lint and test
        run: make check   # must mirror 04-quality-gates.md
```

Agents copy and adapt YAML examples from `04-quality-gates.md` into the
project repository. Review all workflow changes with a human before merging.

---

## Environment Strategy

| Environment | Purpose | Deployed by | Approval required |
|-------------|---------|-------------|-------------------|
| `dev` | Local development and agent execution | Developer / agent locally | No |
| `scan` | Ephemeral container instance for DAST | CI — spun up and torn down per pipeline run | No |
| `develop` | Pre-release integration image | CI on push to `develop` | No |
| `production` | Live system | CI on merge to `main` via `release/*` PR | Yes (defined in overlay) |

- **DAST runs against an ephemeral containerized instance**, not a persistent
  staging environment. The application image is built, started as a Docker
  container, scanned by ZAP, and stopped — all within the pipeline run. This
  removes the dependency on a live staging environment and ensures every scan
  is against the exact artifact being promoted.
- **`develop` images are pre-release.** They are tagged `develop` and
  `<branch>-<sha>` but never `latest`. They are suitable for internal testing
  but not production promotion.
- **Environment-specific configuration** is injected via GitHub Actions secrets
  and environment variables — never hardcoded or branched in application code.
- **Production deployments are logged.** Every deployment to production records:
  the trigger (who or what), the commit SHA, the image digest, the timestamp,
  and the outcome.

---

## Agent Behavior in Pipelines

- **Agents do not merge their own PRs.** A pipeline passing is necessary but
  not sufficient for merge. Human review is required unless the project overlay
  explicitly defines an auto-merge policy for specific change types (e.g.,
  dependency patch updates).
- **Agents do not modify pipeline configuration** without human review. Changes
  to `.github/workflows/` files require explicit approval.
- **Agents do not suppress pipeline failures.** If a pipeline fails, the agent
  addresses the root cause. Adding `continue-on-error: true` to bypass a
  failing step is not acceptable without human approval and a tracked issue.
- **Agents commit pipeline-friendly code.** Before opening a PR, agents must
  confirm that the code they are committing would pass Stage 1 checks locally
  where tooling is available (lint, type check, unit tests).
