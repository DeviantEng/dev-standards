# Git Workflow

This document defines commit conventions, branch behavior, PR process, and
agent-specific git rules. It works in conjunction with the branch strategy
defined in `03-cicd.md` and the quality gates in `04-quality-gates.md`.

---

## Commit Conventions

All commits follow the Conventional Commits specification. This enables
automated changelog generation, semantic versioning, and clear history
navigation for both humans and agents.

### Format

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Types

| Type | When to use |
|------|-------------|
| `feat` | A new feature or capability |
| `fix` | A bug fix |
| `security` | A security fix or hardening change (not a bug fix) |
| `perf` | A performance improvement with no behavior change |
| `refactor` | A code change that is neither a fix nor a feature |
| `test` | Adding or correcting tests only |
| `chore` | Build process, dependency updates, tooling changes |
| `docs` | Documentation only changes |
| `ci` | Changes to CI/CD workflow files or pipeline scripts |
| `revert` | Reverting a previous commit |

### Rules

- **Subject line**: 72 characters maximum. Imperative mood. No period at the
  end. `add user authentication` not `added user authentication` or `adds`.
- **Scope**: Optional but encouraged. Names the module, component, or area
  affected. `feat(auth): add OAuth2 login` not `feat: add OAuth2 login`.
- **Body**: Required when the subject line alone does not convey the why.
  Wrap at 72 characters. Explain the motivation and what changed — not how
  (the diff shows that).
- **Footer**: Used for breaking changes and issue references.
  - Breaking changes: `BREAKING CHANGE: <description>`
  - Issue references: `Closes #123` or `Refs #456`
- **One logical change per commit.** A commit that touches authentication,
  database schema, and the frontend is three commits. Keep scope atomic.
- **No `WIP` commits on shared branches.** Work-in-progress commits are
  acceptable on personal feature branches but must be squashed before PR.

### Examples

```
feat(auth): add OAuth2 login via GitHub provider

Replaces the previous username/password flow for developer accounts.
GitHub OAuth is used because all developers already have GitHub accounts
and it removes the need to manage credentials separately.

Closes #42
```

```
fix(clients): handle connection timeout in Spotify client

The client was raising an unhandled AttributeError when the upstream
API returned a 504. Now raises a typed SpotifyTimeoutError that callers
can handle explicitly.

Refs #88
```

```
security(deps): upgrade cryptography to 42.0.5

Addresses CVE-2024-26130. No API changes. Pin updated in requirements.txt
and lockfile regenerated.

Closes #101
```

```
ci: pin all Actions to full commit SHAs

Tags are mutable and can be redirected. Pinning to SHAs ensures the
pipeline runs the exact reviewed version of each Action.
```

---

## Pre-Commit Hooks

Pre-commit hooks run locally before every commit as an early warning layer.
They do not replace CI — CI is the enforcement layer. Hooks catch the fastest,
cheapest failures before they reach the pipeline.

Choose one profile and declare it in the project root `AGENTS.md`.

### Profile A — Full hooks (default for greenfield)

Individual hooks for gitleaks, ruff, and standard file hygiene:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: <pinned-sha>
    hooks:
      - id: gitleaks

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: <pinned-version>
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: <pinned-version>
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-added-large-files
        args: [--maxkb=500]
      - id: no-commit-to-branch
        args: [--branch, main, --branch, develop]
```

### Profile B — Delegate to Makefile

Single hook runs `make check`. Valid only when the Makefile covers every
Stage 1–2 gate the project requires (see `04-quality-gates.md`).

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: make-check
        name: make check
        entry: make check
        language: system
        pass_filenames: false
        always_run: true
```

### Hook Behavior Rules

- **Hooks are not optional.** Every developer and agent working on a project
  must have pre-commit installed and hooks active. Setup instructions belong
  in the project `README.md`.
- **`no-commit-to-branch`** prevents direct commits to `main` and `develop`
  at the local level. Branch protection rules enforce this in GitHub as the
  authoritative layer.
- **Hooks must not be bypassed** with `git commit --no-verify` except in a
  genuine emergency. Any bypass must be documented in the commit message and
  a follow-up issue filed immediately.
- **Agents must run hooks** or replicate their checks before committing.
  If the agent environment does not support pre-commit, the equivalent checks
  (lint, format, secret scan) must be run manually in the pipeline before push.

### Installation

```bash
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg  # enforces commit message format
```

Run against all files on first install to establish a clean baseline:

```bash
pre-commit run --all-files
```

---

## Agent Development Flow

This section describes the end-to-end workflow an AI agent must follow when
given a development task. It applies to all agents regardless of tooling
(Claude Code, Factory, Codex, etc.).

### Step 1 -- Orient Before Acting

Before writing a single line of code:

1. Read the project root `AGENTS.md`
2. Read `./dev-standards/00-overview.md`
3. Read the relevant module(s) for the task (security change? read `02-security.md`)
4. Read any ADRs in `docs/adr/` that cover the area being changed
5. Search the codebase for existing implementations of similar functionality

Do not skip orientation. An agent that starts coding immediately without
reading the standards is operating outside this workflow.

### Step 2 -- Plan and Confirm

For any non-trivial task:

1. Write a brief plan: what will change, which files, what tests will be added
2. Surface the plan to the human before starting if the scope is large or unclear
3. Identify any ambiguities that could lead to a security or architectural
   decision and resolve them before proceeding

### Step 3 -- Branch

```bash
# Always branch from develop, never from main
git checkout develop
git pull origin develop
git checkout -b feature/<ticket-id>-short-description
```

Branch naming follows the convention in `03-cicd.md`. No work is done directly
on `develop` or `main`.

### Step 4 -- Develop Incrementally

- Write tests before or alongside the code, never after
- Commit in small, logical increments following Conventional Commits
- Run pre-commit hooks after each commit
- Do not accumulate a large uncommitted working tree

### Step 5 -- Self-Review Before Push

Before pushing the branch, the agent must verify (substitute `{source_root}` and
`{test_path}` from the project overlay — see `01-coding-standards.md`):

```bash
# All of these must pass locally before push
ruff check .
ruff format --check .
mypy {source_root}/ --strict          # when Python typecheck enabled in overlay
pytest {test_path}/ -v --cov={source_root} --cov-fail-under=<threshold>
pre-commit run --all-files
# Or: make check   (when Makefile is configured per 04-quality-gates.md)
```

If any check fails, fix it before pushing. Do not push a branch that the
agent knows will fail CI.

### Step 6 -- Open PR to `develop`

- PR title follows Conventional Commits format
- Description covers: what changed, why, tests added, any debt or suppressions
- Link the issue
- Assign for human review -- agents do not self-approve
- Update `CHANGELOG.md` as part of this PR (see Changelog Standards below)

### Step 7 -- Wait for CI and Review

- Do not merge until CI passes and a human has approved
- Address review feedback with new commits, not force-push
- If CI fails, diagnose and fix -- do not re-trigger without a code change

### Step 8 -- After Merge

- Delete the feature branch
- Confirm with the human before starting the next task

---

## Pull Request Process

### Opening a PR

Every PR must have:

- **A descriptive title** following Conventional Commits format:
  `feat(auth): add OAuth2 login via GitHub provider`
- **A description** covering:
  - What changed and why (not how — the diff shows that)
  - Any `TODO(debt):` items introduced
  - Any suppression added to Gitleaks, pip-audit, Trivy, or ZAP, with the
    reason
  - Testing approach — what was tested and how
  - Any manual verification steps taken
- **A linked issue.** PRs without a linked issue require an explanation of
  why one does not exist.
- **A passing pipeline.** Do not request review on a PR with a failing
  pipeline unless the failure is a known flaky test (documented in the PR).

### PR Size

- **Small PRs are strongly preferred.** A PR that can be reviewed in under
  15 minutes is a good PR. A PR that requires an hour is a process problem.
- If a task naturally produces a large PR, break it into a stack of smaller
  PRs with a clear dependency order.
- As a rough guide: under 400 lines changed is comfortable, 400–800 requires
  justification, over 800 should be split.

### Review Requirements

- **At least one human review is required** before merge, regardless of
  pipeline status. Agents do not approve their own PRs.
- Reviewers are responsible for:
  - Correctness and logic
  - Adherence to these standards (coding, security, testing)
  - Adequacy of test coverage for the change
  - Validity of any suppressions or debt markers introduced
- **Requested changes must be addressed**, not dismissed. If the reviewer is
  wrong, explain why in a comment — do not simply dismiss and merge.
- **Approvals are invalidated by subsequent pushes** unless the reviewer
  re-approves. GitHub branch protection should enforce this (`Dismiss stale
  pull request approvals when new commits are pushed`).

### Merging

- **Squash merge** is the default for feature and fix branches into `develop`.
  The squash commit message must follow Conventional Commits format and
  summarize the branch intent — not list every commit in the branch.
- **Merge commit** is used for `release/*` PRs into `main` to preserve the
  release history clearly.
- **Never force-push to `main` or `develop`.** Force-push is only acceptable
  on personal feature branches before a PR is opened.
- Delete the source branch after merge. Merged branches left open become
  confusing noise.

---

## Versioning

Projects use **Semantic Versioning** (SemVer): `MAJOR.MINOR.PATCH`.

| Increment | When |
|-----------|------|
| `PATCH` | Bug fixes, security patches, dependency updates with no behavior change |
| `MINOR` | New features that are backward-compatible |
| `MAJOR` | Breaking changes |

- Version is the single source of truth defined in one place per project
  (e.g., `__version__.py` for Python, `package.json` for Node). The publish
  pipeline reads from this file — it does not infer version from git tags.
- Version is bumped in the `release/*` branch as part of the release commit,
  not as a separate follow-up.
- `BREAKING CHANGE:` in a commit footer triggers a MAJOR bump. `feat:` triggers
  MINOR. Everything else triggers PATCH. Automated tooling (e.g., `semantic-release`)
  may be used to enforce this — defined in the project overlay.

---

## Agent Git Behavior

Agents operating in a project repository must follow these rules without
exception:

- **Branch before touching code.** Agents always create a feature branch from
  `develop` before making changes. No work is done on `develop` or `main`
  directly.
- **Commit messages follow Conventional Commits.** Agents must produce
  well-formed commit messages — not `fix stuff`, `update`, or `changes`.
- **One logical change per commit.** Agents must not bundle unrelated changes
  into a single commit for convenience.
- **Run pre-commit checks before pushing.** Agents must verify that lint,
  format, and secret scan pass before pushing a branch.
- **Do not amend or force-push shared branches.** Once a branch has been
  pushed and a PR opened, history is not rewritten. New commits are added
  to address review feedback.
- **Do not commit generated files that belong in `.gitignore`.** Build
  artifacts, compiled assets, virtual environments, and tool caches are
  never committed.
- **Surface uncertainty before committing.** If an agent is unsure whether
  a change is correct, complete, or consistent with the standards, it flags
  the uncertainty in a PR comment rather than committing with low confidence
  and hoping review catches it.
- **Never use `git push --force` on any branch with an open PR.** This
  rewrites history that reviewers have already read. Use `git push` with new
  commits only.

---

## Changelog Standards

Every project maintains a `CHANGELOG.md` in the project root following the
[Keep a Changelog](https://keepachangelog.com) format. The changelog is
written for humans -- it describes what changed and why it matters, not a
dump of commit messages.

### Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.
The format is based on Keep a Changelog and this project uses Semantic Versioning.

## [Unreleased]

### Added
- Description of new functionality added

### Changed
- Description of changes to existing functionality

### Fixed
- Description of bugs fixed

### Security
- Description of security fixes or dependency upgrades addressing CVEs

### Removed
- Description of removed features or deprecated functionality

## [1.2.0] - 2026-03-15

### Added
- OAuth2 login via GitHub provider for developer accounts

### Fixed
- Connection timeout in Spotify client now raises typed SpotifyTimeoutError

[Unreleased]: https://github.com/org/project/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/org/project/compare/v1.1.0...v1.2.0
```

### Rules

- **`[Unreleased]`** is always the top section. New changes go here during
  development. It is never left empty in a release -- if there is nothing
  notable, that is a signal to question whether a release is warranted.
- **Categories used**: Added, Changed, Fixed, Security, Removed, Deprecated.
  Do not invent new categories. Not every category is required in every release.
- **One entry per meaningful change**, not per commit. A bug fix that took
  three commits is one changelog entry describing the fix, not three entries.
- **Written for users and reviewers**, not developers. Avoid internal
  implementation detail. Describe the observable effect of the change.
- **Security entries are always included** when a CVE is addressed, even for
  transitive dependency updates. Include the CVE ID and affected package.
- **Agents update the changelog as part of the feature PR**, not as a
  separate follow-up. The `[Unreleased]` section is updated in the same PR
  as the code change.
- **Comparison links at the bottom** are updated on every release. They allow
  readers to see exactly what changed between versions on GitHub.

---

## Release Process

A release is a deliberate, human-initiated event. Agents support the process
but do not initiate or complete a release without explicit human instruction.

### When to Release

A release is appropriate when:
- One or more features in `[Unreleased]` are ready for production
- A security fix must be shipped immediately
- A breaking change is being promoted

`develop` must be passing CI fully before a release branch is cut.

### Step 1 -- Determine the Version Bump

Review the `[Unreleased]` section of `CHANGELOG.md` and the commits on
`develop` since the last release:

| If `[Unreleased]` contains | Version bump |
|----------------------------|-------------|
| Any `BREAKING CHANGE` footer | MAJOR |
| Any `feat` with no breaking change | MINOR |
| Only `fix`, `security`, `chore`, `docs`, `ci` | PATCH |

### Step 2 -- Cut the Release Branch

```bash
git checkout develop
git pull origin develop
git checkout -b release/<version>
# e.g. release/1.3.0
```

### Step 3 -- Update Version and Changelog

In the release branch, make exactly two changes:

1. **Bump the version** in the project version file (`__version__.py`,
   `package.json`, etc.) to the new version number.

2. **Finalize the changelog**: rename `[Unreleased]` to `[<version>] - <date>`,
   add a new empty `[Unreleased]` section at the top, and add the comparison
   link at the bottom.

```bash
git add CHANGELOG.md <version-file>
git commit -m "chore(release): prepare v<version>"
git push origin release/<version>
```

No other changes belong in a release branch. If a bug is found during release
preparation, fix it on a feature branch against `develop` first, then rebase
or merge `develop` into the release branch.

### Step 4 -- Open PR to `main`

Open a PR from `release/<version>` to `main`. The PR triggers the full
`pr-checks.yml` pipeline including ZAP full scan. The PR description should
include the changelog section being released, copied verbatim.

**The PR cannot merge until:**
- All CI checks pass (lint, type, tests, SAST, SCA, ZAP full scan)
- At least one human has approved
- No outstanding review comments

### Step 5 -- Merge and Tag

Merge using a **merge commit** (not squash) to preserve the release boundary
in history. Immediately after merge, create and push an annotated tag:

```bash
git checkout main
git pull origin main
git tag -a v<version> -m "Release v<version>"
git push origin v<version>
```

The tag triggers the `publish.yml` pipeline, which builds the multi-arch
image, runs Trivy and ZAP baseline, and pushes to the registry with semantic
version tags (`latest`, `v1`, `v1.3`, `v1.3.0`).

### Step 6 -- Merge `main` Back to `develop`

After tagging, merge `main` back into `develop` to keep histories aligned:

```bash
git checkout develop
git pull origin develop
git merge main
git push origin develop
```

### Step 7 -- Delete the Release Branch

```bash
git push origin --delete release/<version>
```

---

## README Standards

Every project must have a `README.md` in the project root. It is the first
thing a developer or agent reads when approaching the project. It must be
useful without being exhausting.

### Philosophy

A README should feel like it was written by a person who wants to help, not
by a committee trying to document everything. It answers the questions a new
contributor actually has in the order they have them: what is this, how do I
run it, how do I contribute?

- **No excessive emoji.** One or two is fine if the project tone warrants it.
  A page full of emoji makes documentation look unserious and is harder to
  scan.
- **No walls of text.** If a section is growing beyond a few paragraphs,
  it probably belongs in `docs/` or a `README-extended.md`, not the root README.
- **Write for humans first.** Agents can read anything. Humans skim. Optimize
  for skimmability -- short paragraphs, clear headings, code blocks for
  anything that must be typed exactly.
- **Stay current.** A README with outdated install instructions is worse than
  no README. If a step changes, the README changes in the same PR.

### Required Sections (in this order)

```markdown
# Project Name

One or two sentences describing what this project does and who it is for.
No buzzwords. Just what it is.

## Getting Started

The minimum steps to get a working local instance. Assume the reader has
the language runtime installed. Do not explain what Python is.

## Configuration

What environment variables are required. Point to `.env.example`. Do not
list every variable -- just the ones needed to get started.

## Running Tests

The exact command to run the test suite. One command, copy-pasteable.

## Contributing

How to contribute: branch naming, PR process, where to find the standards.
One short paragraph pointing to `AGENTS.md` and `dev-standards/` for detail.
```

### What Belongs in `README-extended.md`

Recommended for human deep documentation that would make the root README too
long to skim. The root README links to it with one line:
`For more detail, see README-extended.md.`

Include:

- Detailed architecture explanation
- Full environment variable reference
- Deployment runbook and **Development Flow** (branch/gate table — keep current)
- Troubleshooting guide
- Design decisions that do not warrant a full ADR
- Pointers to `docs/security-audit-followups.md` and feature specs

Do not duplicate dev-standards module content — link to the submodule instead.
When submodule standards change, update extended docs in the same project change.

### What Does Not Belong in Any README

- Step-by-step tutorials for the tools used (link to their docs instead)
- Full API reference (use generated docs or `docs/`)
- Changelog content (that lives in `CHANGELOG.md`)
- Security vulnerability disclosure instructions (use `SECURITY.md`)

---

## `.gitignore` Standards

Every project must include a `.gitignore` that covers at minimum:

```gitignore
# Environment and secrets
.env
.env.*
!.env.example

# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
dist/
build/
.mypy_cache/
.ruff_cache/
.pytest_cache/
htmlcov/
*.coverage
coverage.xml

# Node
node_modules/
dist/
.next/
.nuxt/

# Editors and OS
.DS_Store
.idea/
.vscode/
*.swp
*.swo

# Docker
*.tar

# Trivy and scan artifacts
trivy-results.*
zap-report.*
```

- `.env.example` is always committed with dummy values and comments explaining
  each variable. It is the documentation for required configuration.
- Secrets in `.env` are never committed. The pre-commit hook enforces this via
  Gitleaks. `.gitignore` is a convenience layer, not the security layer.
