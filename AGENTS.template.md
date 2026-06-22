# [Project Name] — Agent Entry Point

> **Generation:** This file is produced at the **project root** from
> `dev-standards/AGENTS.template.md` + a project plan file. See
> `dev-standards/06-agent-guidance.md` and `dev-standards/00-overview.md`.
> There is no `AGENTS.md` inside `dev-standards/`.

> **This file is public.** Work through the safety checklist at the bottom
> before committing. See `dev-standards/02-security.md` (Public Repository
> Safety). Replace sensitive agent output with:
> `<!-- HUMAN: complete this section offline; do not include sensitive details -->`

Agents start here, then read `./dev-standards/` modules as needed. Shared
standards are non-negotiable — this file may tighten them, never loosen
security rules in `02-security.md`.

---

## Project Identity

| Field | Value |
|-------|-------|
| **Name** | [Project Name] |
| **Purpose** | [One or two sentences] |
| **Repository** | [GitHub URL] |
| **Stack** | [e.g. Python 3.14, TypeScript, FastAPI] |
| **Container registry** | [e.g. ghcr.io/org/project] |

---

## Profiles

| Profile | Value |
|---------|-------|
| **Layout** | [src-layout / app-root-layout — see `01-coding-standards.md`] |
| **Team workflow** | [enterprise / solo-small-team — see `03-cicd.md`] |
| **CI triggers** | [release-gate / integration-gate — see `03-cicd.md`] |

### Paths (from layout profile)

| Placeholder | This project |
|-------------|--------------|
| `{source_root}` | [e.g. `src/` or `app/`] |
| `{test_path}` | [e.g. `tests/unit/` or `tests/`] |
| `{cov_threshold}` | [e.g. 80] |

### Layout deviations

<!-- Only document differences from the standard profile skeleton — not a full tree. -->

- [e.g. `commands/` at repo root for CLI entrypoints]

---

## Commands

Use these exact commands. Gate details: `dev-standards/04-quality-gates.md`.

```bash
make check          # local gate SSOT — must mirror CI
make fix            # auto-fix where supported
# [Add non-obvious setup: env copy, build order, docker, migrations]
```

---

## Domain Glossary

| Term | Meaning in this project |
|------|-------------------------|
| [Term] | [Definition] |

---

## Areas Requiring Extra Care

| Area | Why | What to do |
|------|-----|------------|
| [e.g. clients/] | [Rate limits] | [Check retry logic first] |

---

## Feature Specs

Link cross-cutting docs in `docs/` for specific work — do not duplicate them.

| Change type | Read first |
|-------------|------------|
| [e.g. Unit tests] | [docs/testing_unit_spec.md] |

---

## Agent Boundaries

### Always

- Read this file and the relevant `dev-standards/` module before coding
- Run `make check` (or equivalent) before push
- Tests in the same change as the code they cover

### Ask first

- Workflow changes, dependency major bumps, security suppressions
- Schema migrations, submodule bumps with breaking gate changes

### Never

- Commit secrets; bypass CI; edit `dev-standards/` submodule
- Force-push protected branches; push directly to `main`

---

## Standards Reference

| Module | When |
|--------|------|
| `./dev-standards/00-overview.md` | Setup, agent rules, definition of done |
| `./dev-standards/01-coding-standards.md` | Writing code |
| `./dev-standards/02-security.md` | Auth, data, secrets, deps |
| `./dev-standards/03-cicd.md` | Pipeline and branch behavior |
| `./dev-standards/04-quality-gates.md` | Tools, thresholds, setup |
| `./dev-standards/05-git-workflow.md` | Commits and PRs |
| `./dev-standards/06-agent-guidance.md` | Writing and updating this file |

---

## Pre-Commit Safety Checklist

Complete before committing this file. This file is public.

- [ ] No passwords, tokens, API keys, or credentials anywhere in this file
- [ ] No internal hostnames, IP addresses, or private URLs
- [ ] Environment variable table lists names and purposes only — not values
- [ ] External dependencies table lists service names and paths only — not endpoints or auth
- [ ] No employee names, emails, or personal contact information
- [ ] No customer names, customer data, or PII
- [ ] No internal infrastructure topology or private system names
- [ ] No known vulnerability details of production systems
- [ ] No proprietary business logic or trade secrets
- [ ] Incomplete sections marked with `<!-- HUMAN: ... -->` placeholders
- [ ] Known accepted risks table contains only publicly safe information

When all boxes are checked, this file is safe to commit.

---

## Changelog

| Date | Change | Author |
|------|--------|--------|
| [YYYY-MM-DD] | Initial overlay created | [Name / agent] |
