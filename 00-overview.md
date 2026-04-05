# DevEng Development Standards — Overview

This repository is consumed as a git submodule in project repositories. It
defines shared standards, workflows, and quality gates that apply across all
projects. It is not the agent entry point — agents start at the project root
`AGENTS.md`, which references this file.

---

## How This Works

```
/opt/projects/project1/
├── AGENTS.md              ← agent entry point (project-specific, references this repo)
└── dev-standards/         ← this repo, as a git submodule
    ├── 00-overview.md     ← you are here
    ├── 01-coding-standards.md
    ├── 02-security.md
    ├── 03-cicd.md
    ├── 04-quality-gates.md
    ├── 05-git-workflow.md
    └── AGENTS.template.md ← scaffold for creating the project AGENTS.md
```

Agents auto-discover `/opt/projects/project1/AGENTS.md` as their entry point.
That file references `./dev-standards/00-overview.md` (this file) and the
relevant modules below. Agents never navigate here directly — they are pointed
here by the project `AGENTS.md`.

---

## Module Index

| Module | File | Purpose |
|--------|------|---------|
| Overview | `00-overview.md` | This file — structure, setup, how to use |
| Coding Standards | `01-coding-standards.md` | Modularity, reuse, naming, error handling, tech debt, project structure |
| Security | `02-security.md` | Secure-by-default rules, secrets, scanning requirements, agent security rules |
| CI/CD Pipeline | `03-cicd.md` | Branch strategy, release model, pipeline stages, GitHub Actions standards |
| Quality Gates | `04-quality-gates.md` | Tooling, commands, thresholds — what must pass and how |
| Git Workflow | `05-git-workflow.md` | Commit conventions, PR process, pre-commit hooks, agent git behavior |
| Project Template | `AGENTS.template.md` | Scaffold for creating a new project's `AGENTS.md` |

---

## Setting Up a New Project

### Step 1 — Add this repo as a submodule

```bash
cd /opt/projects/your-new-project
git submodule add git@github.com:<your-org>/dev-standards.git dev-standards
```

### Step 2 — Create the project `AGENTS.md` using AI

Do not fill in `AGENTS.template.md` manually line by line. Use an AI agent to
generate a complete, accurate project overlay in one pass.

Copy the template to the project root:

```bash
cp dev-standards/AGENTS.template.md ./AGENTS.md
```

Then prompt your AI agent with something like:

```
I have a new project called [Project Name]. Here is its purpose: [description].
The stack is [language/framework]. It connects to [external services].
The GitHub repo is [URL]. The container registry is [URL].

Using AGENTS.md as the template, fill in every section with accurate,
project-specific information. Reference ./dev-standards/ modules where
indicated. Do not leave any placeholder text — make every section complete
and correct for this project.
```

The agent will produce a complete, immediately usable `AGENTS.md` that serves
as the entry point for all future agent sessions on this project.

### Step 3 — Review and commit

> **Before committing: this file will be public.** Treat it as if it is
> already visible on GitHub. It must contain no secrets, credentials, internal
> hostnames, private URLs, employee contact details, or any information that
> should not be publicly readable. See `02-security.md` -- Public Repository
> Safety for the full list of what must never appear here.

Review the generated `AGENTS.md` for accuracy and safety:

- Domain concepts and their definitions (names of concepts, not sensitive data)
- External dependency list: service names and env var names only -- not credentials or URLs
- Environment variable table: variable names and purpose only -- never values
- Commands reference (test, lint, build, run)
- Any known accepted risks or data classification (classification labels, not data itself)
- Confirm no internal infrastructure details, hostnames, or IP addresses crept in
- Confirm no credentials, tokens, or secrets appear anywhere in the file
- Replace any sensitive content the agent included with a placeholder comment:
  `<!-- HUMAN: complete this section offline; do not include sensitive details -->`

Then commit:

```bash
git add AGENTS.md dev-standards
git commit -m "chore: add dev-standards submodule and project AGENTS.md"
git push
```

### Step 4 — Install pre-commit hooks

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

Standards are updated via PR against this repo — never committed directly.

To adopt an update in a consuming project:

```bash
cd /opt/projects/your-project
git submodule update --remote dev-standards
git add dev-standards
git commit -m "chore: bump dev-standards to <short-sha>"
git push
```

Projects pin to a specific commit and opt in to updates deliberately.
Review security rule and pipeline behavior changes carefully before bumping.

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
- **Never** push directly to `main`, `master`, or `develop`. All changes go
  through a PR.
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

A task is not complete until all of the following pass. Full details in
`04-quality-gates.md`.

- [ ] Linting passes with no undocumented suppressions
- [ ] Type checking passes
- [ ] Unit tests pass with coverage threshold met
- [ ] Tests written in the same PR as the code they cover
- [ ] Dependency vulnerability scan clean (no HIGH or CRITICAL unmitigated)
- [ ] Container image Trivy scan clean (no HIGH or CRITICAL unmitigated)
- [ ] DAST (ZAP) scan passes against containerized artifact
- [ ] No secrets detected in diff (pre-commit hook + CI)
- [ ] PR has a meaningful description, linked issue, and human review
