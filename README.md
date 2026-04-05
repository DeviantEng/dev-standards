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
then has its own `AGENTS.md` at the root -- the entry point for any agent or
developer -- which references these shared docs for the rules that apply
everywhere, and adds project-specific context on top.

```
your-project/
├── AGENTS.md        <- start here; project-specific + references dev-standards
└── dev-standards/   <- this repo
    ├── 00-overview.md
    ├── 01-coding-standards.md
    ├── 02-security.md
    ├── 03-cicd.md
    ├── 04-quality-gates.md
    ├── 05-git-workflow.md
    └── AGENTS.template.md
```

## Adding to a Project

```bash
cd /opt/projects/your-project
git submodule add git@github.com:<your-org>/dev-standards.git dev-standards
cp dev-standards/AGENTS.template.md ./AGENTS.md
```

Then use an AI agent to fill in `AGENTS.md` with project-specific details.
The template has prompting guidance at the top, and `00-overview.md` walks
through the full setup process.

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
| `01-coding-standards.md` | Modularity, naming, error handling, testing, project structure |
| `02-security.md` | Security requirements, secrets handling, scanning coverage |
| `03-cicd.md` | Branch strategy, pipeline stages, GitHub Actions standards |
| `04-quality-gates.md` | Specific tools, commands, and thresholds for every gate |
| `05-git-workflow.md` | Commits, PRs, changelog, release process, README standards |
| `AGENTS.template.md` | Template for a project's root AGENTS.md |

## Updating the Standards

Changes to these standards go through a PR against this repo. Consuming
projects pin to a specific commit and opt in to updates deliberately by
bumping the submodule reference -- they do not get updates automatically.

To update a project to a newer version of these standards:

```bash
cd /opt/projects/your-project
git submodule update --remote dev-standards
git add dev-standards
git commit -m "chore: bump dev-standards to <short-sha>"
git push
```

Review the changes before bumping, particularly anything in `02-security.md`
or `04-quality-gates.md` that might affect a passing pipeline.

## Contributing

Changes follow the same workflow described in `05-git-workflow.md`. Open a
feature branch, make the change, open a PR. Security rule changes require a
changelog entry and a minor version bump. Breaking pipeline changes require
a major version bump.
