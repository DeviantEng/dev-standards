# [Project Name] — Agent Entry Point

> **Setup note**: This file was generated from `dev-standards/AGENTS.template.md`.
> Review every section for accuracy before committing. See
> `dev-standards/00-overview.md` for full setup instructions.

> **This file is public.** It will be committed to a repository that may be
> publicly visible on GitHub or shared with anyone who has access to the
> codebase. It must contain no secrets, credentials, tokens, internal hostnames,
> private URLs, employee contact details, customer data, or any information that
> should not be publicly readable. Before committing, work through the safety
> checklist at the bottom of this file. See `dev-standards/02-security.md`
> (Public Repository Safety) for the complete list of what must never appear here.
> If the AI agent included anything sensitive, replace it with:
> `<!-- HUMAN: complete this section offline; do not include sensitive details -->`

This is the agent entry point for **[Project Name]**. Agents and developers
start here, then follow references into `./dev-standards/` for shared standards.

**Shared standards are non-negotiable.** This file may extend or tighten them
but never loosens security rules, quality gate thresholds, or agent behavioral
constraints defined in `dev-standards/`.

## Standards Reference

Read the relevant module before starting any task:

| Module | When to read it |
|--------|----------------|
| `./dev-standards/00-overview.md` | Always — start here for shared standards |
| `./dev-standards/01-coding-standards.md` | Before writing any code |
| `./dev-standards/02-security.md` | Before any change touching auth, data, secrets, or deps |
| `./dev-standards/03-cicd.md` | Before modifying pipeline or branch behavior |
| `./dev-standards/04-quality-gates.md` | Before opening a PR — confirms definition of done |
| `./dev-standards/05-git-workflow.md` | Before committing or opening a PR |

---

## Project Overview

<!-- Replace this section with a concise description of the project. -->

| Field | Value |
|-------|-------|
| **Project name** | [Project Name] |
| **Purpose** | [One or two sentences describing what this project does] |
| **Owner / team** | [Team name or individual] |
| **Repository** | [GitHub URL] |
| **Container registry** | [Registry URL, e.g. ghcr.io/org/project] |
| **Primary language(s)** | [e.g. Python 3.14, TypeScript] |
| **Status** | [Active / Maintenance / Deprecated] |

---

## Quick Start

<!-- Commands a developer or agent needs to get a working local environment. -->

```bash
# Clone with submodules
git clone --recurse-submodules <repo-url>
cd <project>

# Install pre-commit hooks (required)
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg

# [Add environment setup steps here]
# e.g. copy .env.example to .env and fill in values
cp .env.example .env

# [Add dependency install steps here]
# e.g. pip install -r requirements.txt -r requirements-dev.txt
#      npm ci --prefix frontend
```

---

## Project Structure

This project follows the universal structure defined in
`dev-standards/01-coding-standards.md`. Project-specific additions are noted
below.

```
[project-root]/
├── AGENTS.md                  ← This file
├── dev-standards/             ← Git submodule (do not edit)
├── README.md
├── src/
│   ├── clients/               ← [Describe external services connected here]
│   ├── models/                ← [Describe primary domain models]
│   ├── services/              ← [Describe primary business logic areas]
│   ├── utils/                 ← Pure helpers
│   └── config/                ← All config loaded here; no env vars elsewhere
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── [frontend/]                ← [If applicable: describe frontend structure]
├── scripts/
├── docs/
│   └── adr/                   ← Architecture Decision Records
└── .github/
    ├── workflows/
    │   ├── pr-checks.yml
    │   ├── publish.yml
    │   └── integration.yml    ← [If applicable]
    └── scripts/
        ├── zap-full-scan.sh
        └── zap-scan.sh
```

<!-- Add or remove directories as appropriate. Describe what lives where. -->

---

## Language and Stack

<!-- Fill in the specific stack for this project. -->

### Python

| Setting | Value |
|---------|-------|
| Version | [e.g. 3.14] |
| Dependency manager | [pip / poetry / uv] |
| Requirements files | [e.g. requirements.txt, requirements-dev.txt] |
| Linter / formatter | Ruff (configured in `pyproject.toml`) |
| Type checker | mypy strict (configured in `pyproject.toml`) |
| Test runner | pytest |
| Coverage minimum | [e.g. 80% — raise above shared minimum if appropriate] |

### JavaScript / TypeScript (if applicable)

| Setting | Value |
|---------|-------|
| Runtime | [Node version] |
| Package manager | npm / pnpm / yarn |
| Framework | [e.g. React, Next.js, Vue] |
| Linter | ESLint with eslint-plugin-security |
| Formatter | Prettier |
| Type checker | tsc --noEmit, strict: true |
| Test runner | [e.g. Vitest, Jest] |

### Additional Languages (if applicable)

<!-- Add a table for each additional language used in the project. -->

---

## Environment Variables

All required environment variables are documented in `.env.example`. No
variable should be added to the codebase without a corresponding entry in
`.env.example` with a comment explaining its purpose and acceptable values.

<!-- List the variable categories here so agents know what to expect. -->

| Category | Variables | Source in production |
|----------|-----------|---------------------|
| [e.g. Database] | [e.g. DB_URL, DB_NAME] | [e.g. GitHub Actions secret] |
| [e.g. External API] | [e.g. SPOTIFY_CLIENT_ID] | [e.g. GitHub Actions secret] |
| [e.g. App config] | [e.g. LOG_LEVEL, PORT] | [e.g. Environment variable] |

---

## Architecture and Domain Concepts

<!-- This section is critical for agents. Describe the domain model and key
concepts so agents do not have to infer them from the code. Stable domain
concepts are safer to document than file paths, which change. -->

### Key Domain Concepts

| Term | Definition |
|------|------------|
| [Term] | [What it means in this project's context] |
| [Term] | [What it means in this project's context] |

### External Dependencies

| Dependency | Purpose | Client location |
|------------|---------|-----------------|
| [e.g. Spotify API] | [e.g. Fetch track metadata] | [e.g. src/clients/spotify.py] |
| [e.g. PostgreSQL] | [e.g. Primary data store] | [e.g. src/clients/db.py] |

### Architecture Decisions

Non-obvious technical decisions are recorded as ADRs in `docs/adr/`. Agents
must read relevant ADRs before making changes to areas they cover.

| ADR | Decision |
|-----|----------|
| [e.g. docs/adr/0001-use-ruff-over-black.md] | [One-line summary] |

---

## Commands Reference

Agents must use these exact commands. Do not invent alternatives.

### Development

```bash
# Run linter
[e.g. ruff check .]

# Run formatter
[e.g. ruff format .]

# Run type checker
[e.g. mypy src/ --strict]

# Run unit tests
[e.g. pytest tests/unit/ -v --cov=src --cov-fail-under=80]

# Run integration tests
[e.g. pytest tests/integration/ -v]

# Run all pre-commit hooks manually
pre-commit run --all-files
```

### Docker

```bash
# Build image
[e.g. docker build -t project:dev .]

# Run container locally
[e.g. docker run --env-file .env -p 8080:8080 project:dev]

# Run Trivy scan locally
trivy image --severity CRITICAL,HIGH project:dev
```

### Database / Migrations (if applicable)

```bash
# [Add migration commands here]
```

---

## Testing Approach

<!-- Describe the testing strategy specific to this project. -->

### Test Co-location Convention

This project uses: **[co-located / mirrored]** test structure.

- [Co-located]: Test files live alongside source files —
  `src/auth/login.py` → `src/auth/test_login.py`
- [Mirrored]: Test files mirror the source tree —
  `src/auth/login.py` → `tests/unit/auth/test_login.py`

### Fixtures and Test Data

- Shared fixtures live in `tests/fixtures/`.
- [Describe how test data is managed — factory functions, fixture files, etc.]
- Integration tests use [describe environment — Docker Compose, test DB, mocks].

### Coverage Threshold

Minimum coverage for this project: **[X]%** (shared minimum is 80%).

---

## CI/CD Pipeline

This project follows the pipeline defined in `dev-standards/03-cicd.md` with
the following project-specific details.

### Workflow Triggers

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `pr-checks.yml` | PR to `main` | Lint, type, tests, SAST, SCA, ZAP full scan |
| `publish.yml` | Push to `main` or `develop` | Build, Trivy scan, ZAP baseline, push to registry |
| `integration.yml` | PR to `main`, push to `main` | Integration tests (if applicable) |

### Image Tagging

| Branch | Tags applied |
|--------|-------------|
| `main` | `latest`, `v<major>`, `v<major.minor>`, `v<major.minor.patch>`, `<branch>-<sha>` |
| `develop` | `develop`, `<branch>-<sha>` |

### Secrets Required

<!-- List the GitHub Actions secrets this project requires. -->

| Secret name | Purpose |
|-------------|---------|
| `GITHUB_TOKEN` | Push to GHCR (automatic) |
| [e.g. `SPOTIFY_CLIENT_SECRET`] | [Purpose] |

---

## Security Notes

<!-- Document project-specific security context beyond the shared standards. -->

### Data Classification

| Data type | Classification | Notes |
|-----------|----------------|-------|
| [e.g. User email] | PII | [How it is handled, stored, masked] |
| [e.g. API tokens] | Confidential | [Where stored, rotation policy] |

### Known Accepted Risks

<!-- Document any accepted risks with full context. -->

| Finding | Tool | Reason accepted | Owner | Review date |
|---------|------|-----------------|-------|-------------|
| [CVE or rule ID] | [Trivy / pip-audit / ZAP] | [Reason] | [Owner] | [Date] |

### License Policy

This project uses the **[MIT / Apache-2.0 / proprietary]** license.

Blocked dependency licenses: GPL, AGPL, LGPL (shared default).
Additional blocked licenses for this project: [none / list any].

---

## Agent Instructions

<!-- Project-specific behavioral rules for agents. These add to, never
replace, the rules in dev-standards/AGENTS.md. -->

### Scope Boundaries

Agents working on this project should:

- [ ] Read `dev-standards/AGENTS.md` before starting any task
- [ ] Read the relevant `dev-standards/` module for the area being changed
- [ ] Read relevant ADRs in `docs/adr/` before changing areas they cover
- [ ] Run `pre-commit run --all-files` before pushing

### Areas Requiring Extra Care

<!-- Call out areas of the codebase that are sensitive, complex, or have
non-obvious behavior that an agent needs to know before touching them. -->

| Area | Why it needs care | What to do |
|------|-------------------|------------|
| [e.g. src/clients/] | [e.g. Rate limits on external APIs] | [e.g. Always check existing retry logic before adding new calls] |
| [e.g. src/config/] | [e.g. Config is validated at startup; schema changes break startup] | [e.g. Update Pydantic model and .env.example together] |

### Out of Scope

Agents must not make changes to the following without explicit human approval:

- `dev-standards/` submodule contents
- `.github/workflows/` files
- Dependency pinned versions (propose in a PR, do not commit directly)
- [Add any project-specific out-of-scope areas]

---

## Pre-Commit Safety Checklist

Complete this checklist before committing this file to any repository.
This file is public. Treat it accordingly.

- [ ] No passwords, tokens, API keys, or credentials appear anywhere in this file
- [ ] No internal hostnames, IP addresses, or private URLs appear in this file
- [ ] The environment variable table contains variable names and purposes only -- not values
- [ ] The external dependencies table contains service names and module paths only -- not endpoints or auth details
- [ ] No employee names, email addresses, or personal contact information appear in this file
- [ ] No customer names, customer data, or PII appear in this file
- [ ] No internal infrastructure topology, network layout, or private system names appear in this file
- [ ] No known vulnerability details or security weaknesses of production systems appear in this file
- [ ] No proprietary business logic or trade secrets appear in this file
- [ ] Any section the agent could not complete safely is marked with a placeholder comment
- [ ] The known accepted risks table contains only information safe to share publicly

When all boxes are checked, this file is safe to commit.

---

## Changelog

<!-- Track significant changes to this overlay document. -->

| Date | Change | Author |
|------|--------|--------|
| [YYYY-MM-DD] | Initial overlay created | [Name / agent] |
