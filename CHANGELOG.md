# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `06-agent-guidance.md` — inclusion filter, generation workflow, Always/Ask/Never tiers
- Layout profiles (`src-layout`, `app-root-layout`) in `01-coding-standards.md`
- Team workflow profiles and CI trigger profiles in `03-cicd.md`
- Workflow authoring guidance for agents in `03-cicd.md`
- Per-gate setup template structure in `04-quality-gates.md`
- New Project Gate Setup checklist in `00-overview.md`
- `CHANGELOG.md` for submodule version tracking
- Makefile `make check` / `make fix` as local gate SSOT in `04-quality-gates.md`
- Maintenance workflow patterns in `03-cicd.md`
- `docs/security-audit-followups.md` overlay pattern in `02-security.md`
- Expanded Renovate setup in `04-quality-gates.md`
- Explicit `README-extended.md` guidance in `05-git-workflow.md`

### Changed

- Slim `AGENTS.template.md` — project-only overlay; no duplicated module content
- AGENTS.md generation model: template + project plan → project root only
- Ruff policy: project-defined rule set; documented ignores required
- Quality gate paths use `{source_root}` and `{test_path}` placeholders
- Agent behavioral rules: solo-team may push to `develop`; `main` via release PR
- Pre-commit Profile A (full hooks) and Profile B (Makefile delegate)

### Fixed

- Removed broken references to nonexistent `dev-standards/AGENTS.md`
- Removed false claim about reusable workflows in `dev-standards/.github/workflows/`
