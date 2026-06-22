# Agent Guidance

This module defines how to write effective agent context for projects that use
dev-standards as a submodule. It complements `00-overview.md` (setup) and
`02-security.md` (non-negotiable rules).

---

## AGENTS.md Generation Model

Project `AGENTS.md` lives at the **project root only**. It is never created
inside `dev-standards/`.

**Inputs:**
- `dev-standards/AGENTS.template.md` — structure and required sections
- **Project plan file** — purpose, stack, constraints (`docs/project-plan.md`,
  README, architecture brief, or equivalent)
- Codebase scan (for existing projects) — paths, commands, layout deviations

**Output:**
- `./AGENTS.md` at project root — links to dev-standards modules, does not
  duplicate their content

**After submodule bump:** review changes in `02-security.md` and
`04-quality-gates.md`. Amend project `AGENTS.md` if overlay facts (paths,
commands, profiles) changed. Never edit the submodule from a consuming project.

---

## Inclusion Filter

Before adding content to `AGENTS.md`, ask:

> Can the agent discover this by reading the code?

- **Yes** → omit from overlay (or one-line pointer to where it lives)
- **No, and it's safety-critical** → include (domain glossary, care areas, env var *names*)
- **No, and it's operational** → include (exact commands, layout profile, `{source_root}`)

**Prefer fixing code over adding instructions.** Clear names, linters, tests,
and structure reduce the need for encyclopedic context files.

---

## What Belongs in the Overlay

| Include | Omit (link instead) |
|---------|---------------------|
| Project identity (name, purpose, repo URL) | Full CI job tables → `03-cicd.md` |
| Layout profile + deviations | Generic directory trees → `01-coding-standards.md` |
| `{source_root}`, `{test_path}`, coverage scope | Gate tooling details → `04-quality-gates.md` |
| Non-obvious commands (`make check`, build order) | Security principles → `02-security.md` |
| Domain glossary | Commit/PR conventions → `05-git-workflow.md` |
| Areas requiring extra care | Module index (pointer to `00-overview.md` is enough) |
| Workflow profile (enterprise vs solo) | |
| Always / Ask / Never boundaries (project-specific additions) | |
| Links to feature specs in `docs/` for specific change types | |

---

## Always / Ask First / Never

These tiers supplement the hard rules in `00-overview.md` and `02-security.md`.

### Always

- Read project root `AGENTS.md` before starting
- Read the relevant `dev-standards/` module for the task area
- Run local gates (`make check` or equivalent) before push
- Write tests in the same change as the code they cover
- Work through the public-safety checklist before committing `AGENTS.md`

### Ask first

- Changes to `.github/workflows/`
- Dependency major version bumps
- Security scan suppressions
- Schema or API breaking changes
- Editing anything inside `dev-standards/` submodule

### Never

- Commit secrets, tokens, or credentials
- Bypass or disable CI gates without human approval
- Create or reference `dev-standards/AGENTS.md` (does not exist)
- Generate architecture dumps into `AGENTS.md` without human review
- Push directly to `main`

---

## Nested AGENTS.md (optional)

Monorepos may use nested `AGENTS.md` files in subpackages for package-specific
context. The **project root** `AGENTS.md` remains the cross-tool SSOT and must
link to nested files. Nested files follow the same inclusion filter — minimal,
non-redundant, non-inferable facts only.

---

## Tool-Specific Overlays

`.cursor/rules/` (or equivalent) may hold glob-scoped, tool-specific rules.
These supplement — never replace — project root `AGENTS.md`. Keep tool rules
narrow; keep cross-tool conventions in `AGENTS.md`.

---

## Generation Workflow

1. Gather inputs: template, project plan, optional codebase scan
2. Generate draft `AGENTS.md` at project root using inclusion filter
3. Human reviews for accuracy and **public safety** (`02-security.md`)
4. Commit `AGENTS.md` and submodule reference together
5. Enable quality gates per `00-overview.md` Step 4

Avoid `/init`-style auto-generation that dumps full architecture into the
overlay. Shorter, precise files outperform encyclopedic standards in the
agent context budget.

---

## Task Sizing

- **Trivial** (typo, single-line fix): proceed; still run gates
- **Non-trivial**: write a brief plan; confirm with human when scope or
  security implications are unclear
- **Small PRs**: prefer changes reviewable in under 15 minutes
- **Surface uncertainty** before commit — flag low-confidence work in PR
  comments, not silent commits

---

## Feature Specs in docs/

Cross-cutting specifications (testing conventions, UI patterns, API contracts)
belong in `docs/`. Link them from `AGENTS.md` for the change types that
require them — e.g. "UI work: read `docs/ui-patterns.md`". Do not copy spec
content into the overlay.

---

## Example Generation Prompt

```
Read dev-standards/AGENTS.template.md and dev-standards/06-agent-guidance.md.
Read my project plan at docs/project-plan.md.

Generate ./AGENTS.md at the project root. Include only non-inferable facts.
Declare layout profile, workflow profile, {source_root}, {test_path}, and
exact commands. Link to dev-standards/ modules — do not duplicate them.
Never write AGENTS.md inside dev-standards/.
```
