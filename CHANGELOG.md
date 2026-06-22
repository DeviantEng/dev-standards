# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Layout profiles (`src-layout`, `app-root-layout`) in `01-coding-standards.md`
- Team workflow profiles (enterprise vs solo/small-team) and CI trigger profiles in `03-cicd.md`
- Workflow authoring guidance for agents in `03-cicd.md` (replaces reusable workflow claim)
- Per-gate setup template structure in `04-quality-gates.md`
- New Project Gate Setup checklist in `00-overview.md`
- `CHANGELOG.md` for submodule version tracking
- Module index entries for `06-agent-guidance.md`

### Changed

- AGENTS.md generation model: template + project plan → project root only; no `dev-standards/AGENTS.md`
- Ruff policy: project-defined rule set in `pyproject.toml` instead of `--select ALL` mandate
- Quality gate paths use `{source_root}` and `{test_path}` placeholders from project overlay
- Agent behavioral rules: solo-team may push directly to `develop`; `main` always via release PR
- Definition of Done references overlay-enabled checks (typecheck, coverage scope)

### Fixed

- Removed broken references to nonexistent `dev-standards/AGENTS.md` in `AGENTS.template.md`
- Removed false claim about reusable workflows in `dev-standards/.github/workflows/`
