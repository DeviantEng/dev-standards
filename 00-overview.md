# DevEng Development Standards — Overview

This repository is consumed as a git submodule in project repositories. It
defines shared standards, workflows, and quality gates that apply across all
projects. It is not the agent entry point — agents start at the **project root**
`AGENTS.md`, which references this file.

There is **no** `AGENTS.md` inside `dev-standards/`. Project `AGENTS.md` is
generated at the project root from `AGENTS.template.md` plus a project plan file.

---

## How This Works

```
/opt/projects/project1/
├── AGENTS.md              ← agent entry point (generated; lives at project root)
├── docs/project-plan.md   ← example project plan input (or README, architecture brief)
└── dev-standards/         ← this repo, as a git submodule (read only)
    ├── 00-overview.md     ← you are here
    ├── 01-coding-standards.md
    ├── 02-security.md
    ├── 03-cicd.md
    ├── 04-quality-gates.md
    ├── 05-git-workflow.md
    ├── 06-agent-guidance.md
    └── AGENTS.template.md ← scaffold for generating project AGENTS.md
```

Agents auto-discover `/opt/projects/project1/AGENTS.md` as their entry point.
That file references `./dev-standards/00-overview.md` (this file) and the
relevant modules below. Agents never navigate to `dev-standards/` directly —
they are pointed here by the project `AGENTS.md`.

---

## Module Index

| Module | File | Purpose |
|--------|------|---------|
| Overview | `00-overview.md` | This file — structure, setup, how to use |
| Coding Standards | `01-coding-standards.md` | Modularity, reuse, naming, error handling, tech debt, layout profiles |
| Security | `02-security.md` | Secure-by-default rules, secrets, scanning requirements, agent security rules |
| CI/CD Pipeline | `03-cicd.md` | Branch strategy, pipeline stages, workflow authoring for agents |
| Quality Gates | `04-quality-gates.md` | Tooling, setup steps, commands, thresholds — what must pass and how |
| Git Workflow | `05-git-workflow.md` | Commit conventions, PR process, pre-commit hooks, agent git behavior |
| Agent Guidance | `06-agent-guidance.md` | How to write AGENTS.md overlays; inclusion filter; generation workflow |
| Project Template | `AGENTS.template.md` | Scaffold for generating a new project's root `AGENTS.md` |

---

## Setting Up a New Project

### Step 1 — Add this repo as a submodule

```bash
cd /opt/projects/your-new-project
git submodule add git@github.com:<your-org>/dev-standards.git dev-standards
```

Do not edit files inside `dev-standards/` from the consuming project.

### Step 2 — Generate the project `AGENTS.md`

Do not fill in `AGENTS.template.md` manually line by line. Use an AI agent to
generate a complete project overlay at the **project root**.

**Inputs:**
- `dev-standards/AGENTS.template.md` — structure and required sections
- **Project plan file** — purpose, stack, constraints (e.g. `docs/project-plan.md`,
  README, or architecture brief)

**Output:**
- `./AGENTS.md` at the project root only — never inside `dev-standards/`

Example prompt:

```
Read dev-standards/AGENTS.template.md and dev-standards/06-agent-guidance.md.
Read my project plan at docs/project-plan.md (and scan the codebase if it exists).

Generate ./AGENTS.md at the project root. Include only non-inferable project
facts: identity, layout profile, workflow profile, paths, commands, domain
glossary, and deviations. Link to dev-standards/ modules — do not duplicate
their content. Do not leave placeholder text. Never write AGENTS.md inside
dev-standards/.
```

Review the generated file using the safety checklist in the template before
committing. See `02-security.md` — Public Repository Safety.

Then commit:

```bash
git add AGENTS.md dev-standards
git commit -m "chore: add dev-standards submodule and project AGENTS.md"
git push
```

### Step 3 — Review and commit

> **Before committing: AGENTS.md will be public.** Treat it as if it is
> already visible on GitHub. It must contain no secrets, credentials, internal
> hostnames, private URLs, employee contact details, or any information that
> should not be publicly readable. See `02-security.md` — Public Repository
> Safety for the full list of what must never appear here.

Review the generated `AGENTS.md` for accuracy and safety:

- Domain concepts and their definitions (names of concepts, not sensitive data)
- External dependency list: service names and env var names only — not credentials or URLs
- Environment variable table: variable names and purpose only — never values
- Commands reference (test, lint, build, run)
- Layout profile and `{source_root}` / `{test_path}` declared
- Any known accepted risks or data classification (classification labels, not data itself)
- Confirm no internal infrastructure details, hostnames, or IP addresses crept in
- Confirm no credentials, tokens, or secrets appear anywhere in the file
- Replace any sensitive content the agent included with a placeholder comment:
  `<!-- HUMAN: complete this section offline; do not include sensitive details -->`

### Step 4 — Enable quality gates

Walk through every gate in `04-quality-gates.md` and configure tooling in the
project. Use this checklist for greenfield projects:

- [ ] Ruff lint + format (`pyproject.toml`, Makefile, CI)
- [ ] Python type check — mypy on `{source_root}` (if Python; declare in overlay)
- [ ] ESLint + Prettier + tsc (if JS/TS frontend)
- [ ] Gitleaks — `.gitleaksignore`, pre-commit, CI
- [ ] pytest + coverage on `{source_root}` / `{test_path}`
- [ ] Bandit SAST on `{source_root}`
- [ ] pip-audit / npm audit
- [ ] OSS license scan (pip-licenses, license-checker)
- [ ] Trivy config (IaC) scan
- [ ] Trivy container image scan
- [ ] ZAP full + baseline scripts
- [ ] zizmor workflow audit
- [ ] Renovate or Dependabot for Action SHA and digest pinning
- [ ] GitHub Actions workflows authored per `03-cicd.md`
- [ ] `make check` mirrors CI Stage 1–2 gates

Legacy projects may track gaps in `docs/compliance-gaps.md` while aligning.
Production promotion still requires all categories in `02-security.md`.

### Step 5 — Install pre-commit hooks

```bash
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg
pre-commit run --all-files
```

---

## Cloning a Project That Uses This Submodule

```bash
# Always use --recurse-submodules
git clone --recurse-submodules git@github.com:<your-org>/project.git

# If you forgot:
git submodule update --init --recursive
```

---

## Updating These Standards

To adopt an update in a consuming project:

```bash
cd /opt/projects/your-project
git submodule update --remote dev-standards
git add dev-standards
git commit -m "chore: bump dev-standards to <short-sha>"
git push
```

Projects pin to a specific commit and opt in to updates deliberately.
After bumping, review changes in `02-security.md` and `04-quality-gates.md`.
Amend the project root `AGENTS.md` if overlay facts (paths, commands, profiles)
need updating. Do not edit the submodule directly.

---

## Core Principles

These apply to every task, in every project, without exception.

### Security First
Never trade security for convenience or speed. If a shortcut introduces a
vulnerability, the shortcut is wrong. See `02-security.md` for specifics.
Security rules in `02-security.md` are not overridable by project overlays.

### No Unnecessary Tech Debt
Every change should leave the codebase in equal or better shape than it was
found. If debt must be incurred, document it inline with a `TODO(debt):` comment
including a tracker link, owner, and expiry. See `01-coding-standards.md`.

### Modularity and Reuse
Before writing new code, check whether the functionality already exists. No
identical block of logic exists in more than one place. See `01-coding-standards.md`.

### Reversibility
Prefer reversible decisions over irreversible ones. Favor feature flags over
hard cutoffs, soft deletes over hard deletes, additive schema changes over
destructive ones.

### Clarity Over Cleverness
Code is read far more than it is written. Prioritize readability. Avoid
abstractions that require the reader to hold too much context in their head.

---

## Agent Behavioral Rules

When operating as an AI coding agent in any project that includes these standards:

- **Always** read the project root `AGENTS.md` before starting any task.
- **Always** read the relevant `dev-standards/` module before writing code.
- **Always** run quality gate checks (see `04-quality-gates.md`) before
  considering a task complete.
- **Never** commit secrets, credentials, tokens, or environment-specific values
  to source control under any circumstances.
- **Never** bypass or disable a linting rule, security scan, or test without
  explicit human approval documented in the PR.
- **Never** push directly to `main` or `master`. All promotion to `main` goes
  through a release PR.
- **Never** push directly to `develop` unless the project overlay declares the
  **solo/small-team** workflow profile (see `03-cicd.md`).
- **Never** modify `dev-standards/` submodule contents from a consuming project.
- **Never** modify `.github/workflows/` files without explicit human approval.
- **Flag and pause** if a task would require violating any rule in `02-security.md`.
  Do not proceed autonomously — surface it for human decision.
- **Prefer small, reviewable commits** over large sweeping changes.
- **Document decisions** in commit messages and PR descriptions. Future agents
  and developers need to understand *why*, not just *what*.
- **Surface uncertainty** before committing. Flag low-confidence changes in a
  PR comment rather than committing and hoping review catches the problem.

---

## Quick Reference: Definition of Done

A task is not complete until all of the following pass for the project's
declared stack and overlay. Full details in `04-quality-gates.md`.

- [ ] Linting passes with no undocumented suppressions
- [ ] Type checking passes (when enabled in overlay for the stack)
- [ ] Unit tests pass with coverage threshold met on declared scope
- [ ] Tests written in the same PR as the code they cover
- [ ] Dependency vulnerability scan clean (no HIGH or CRITICAL unmitigated)
- [ ] Container image Trivy scan clean (no HIGH or CRITICAL unmitigated)
- [ ] DAST (ZAP) scan passes against containerized artifact
- [ ] No secrets detected in diff (pre-commit hook + CI)
- [ ] PR has a meaningful description, linked issue, and human review
