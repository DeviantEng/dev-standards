# DevEng Development Standards

This repository contains the shared development standards used across projects.
It covers coding practices, security requirements, CI/CD pipeline structure,
quality gates, and git workflow conventions.

It is not a standalone project. It is meant to be included in other projects
as a git submodule, where it sits alongside project-specific configuration and
provides a consistent foundation that any developer or AI agent can follow.

## How It Works

When you add this repo as a submodule to a project, your project gets a
`dev-standards/` directory containing all the standards documents. Your project
then has its own `AGENTS.md` at the **project root** — the entry point for any
agent or developer — which references these shared docs and adds project-specific
context. There is no `AGENTS.md` inside the submodule.

```
your-project/
├── AGENTS.md        <- generated entry point; lives at project root
├── docs/            <- project plan or specs (generation input)
└── dev-standards/   <- this repo (read only; do not edit from consuming project)
    ├── 00-overview.md
    ├── 01-coding-standards.md
    ├── 02-security.md
    ├── 03-cicd.md
    ├── 04-quality-gates.md
    ├── 05-git-workflow.md
    ├── 06-agent-guidance.md
    └── AGENTS.template.md
```

## Adding to a Project

```bash
cd /opt/projects/your-project
git submodule add git@github.com:<your-org>/dev-standards.git dev-standards
```

Use an AI agent with `dev-standards/AGENTS.template.md`, your project plan file,
and `dev-standards/06-agent-guidance.md` to generate `./AGENTS.md` at the project
root. See `00-overview.md` for the full setup process and example prompt.

## Cloning a Project That Uses This

```bash
git clone --recurse-submodules git@github.com:<your-org>/your-project.git
```

If you already cloned without the flag:

```bash
git submodule update --init --recursive
```

## What Is in Each Document

| File | What it covers |
|------|----------------|
| `00-overview.md` | How everything fits together; setup instructions |
| `01-coding-standards.md` | Modularity, naming, error handling, testing, layout profiles |
| `02-security.md` | Security requirements, secrets handling, scanning coverage |
| `03-cicd.md` | Branch strategy, pipeline stages, workflow authoring |
| `04-quality-gates.md` | Tools, setup steps, commands, and thresholds for every gate |
| `05-git-workflow.md` | Commits, PRs, changelog, release process, README standards |
| `06-agent-guidance.md` | Writing AGENTS.md overlays; generation workflow |
| `AGENTS.template.md` | Scaffold for generating a project's root `AGENTS.md` |

## Updating the Standards

Consuming projects pin to a specific commit and opt in to updates deliberately by
bumping the submodule reference — they do not get updates automatically.

```bash
cd /opt/projects/your-project
git submodule update --remote dev-standards
git add dev-standards
git commit -m "chore: bump dev-standards to <short-sha>"
git push
```

Review the changes before bumping, particularly anything in `02-security.md`
or `04-quality-gates.md`. Amend the project root `AGENTS.md` if overlay facts
changed. Do not edit the submodule directly.

## Contributing

Changes commit directly to `main` with a `CHANGELOG.md` entry. See
[CONTRIBUTING.md](CONTRIBUTING.md). Consuming projects follow the PR-based
workflow in `05-git-workflow.md`.
