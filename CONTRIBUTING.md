# Contributing to dev-standards

This repository contains shared development standards consumed as a git submodule.
Start with `00-overview.md` to understand how the modules fit together.

## Workflow

This repo commits **directly to `main`**. Pull requests are not used here.

For each change:

1. Make the edit on `main` (or a local branch merged to `main`)
2. Update `CHANGELOG.md` under `[Unreleased]`
3. Commit with a [Conventional Commits](https://www.conventionalcommits.org/) message
4. Push to `main`

Consuming projects follow the PR-based workflow in `05-git-workflow.md`.

## What to change where

| Topic | File |
|-------|------|
| Setup, agent rules, definition of done | `00-overview.md` |
| Code structure, layout profiles | `01-coding-standards.md` |
| Security principles (non-negotiable) | `02-security.md` |
| Pipeline shape, workflow authoring | `03-cicd.md` |
| Tool setup, commands, thresholds | `04-quality-gates.md` |
| Commits, PRs, releases (for consuming projects) | `05-git-workflow.md` |
| AGENTS.md overlay generation | `06-agent-guidance.md` |
| Project overlay template | `AGENTS.template.md` |

## Version notes

Record impact in `CHANGELOG.md`:

- **Security rule changes** — note as minor version impact when tagging
- **Breaking gate or pipeline behavior** — note as major version impact when tagging

Submodule consumers pin to a commit SHA and bump deliberately. Tag releases when
ready; tagging is not required after every change.

## What not to add

- No `AGENTS.md` in this repo — project overlays live at the consuming project root
- No `.github/workflows/` — workflow files belong in consuming projects; this repo
  documents patterns only
- No project-specific configuration

## Review before pushing

- Grep for `dev-standards/AGENTS.md` — that path must not appear
- Grep for `dev-standards/.github/workflows/` — that path must not appear
- Security changes in `02-security.md` affect every consuming project — review carefully
