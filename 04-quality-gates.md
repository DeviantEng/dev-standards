# Quality Gates

This document defines the tooling, setup steps, commands, thresholds, and failure
conditions that fulfill the scanning requirements in `02-security.md` and the
pipeline stages in `03-cicd.md`. Every gate listed here is part of the
**greenfield default**. All must pass before a change is considered shippable.

Tooling choices are recorded here with rationale. If a tool is replaced, update
this document and the project's workflow files in the same change.

---

## Project Path Placeholders

Gate commands use placeholders defined in the project root `AGENTS.md`:

| Placeholder | Meaning | Example (`src-layout`) | Example (`app-root-layout`) |
|-------------|---------|------------------------|----------------------------|
| `{source_root}` | Application source directory | `src/` | `app/` |
| `{test_path}` | Unit test directory | `tests/unit/` | `tests/` |
| `{cov_threshold}` | Minimum line coverage | `80` | project-defined |

Legacy projects aligning to these standards may track gaps in
`docs/compliance-gaps.md` (owner + target date per gate). Production promotion
still requires every category in `02-security.md` to be covered.

---

## Per-Gate Documentation Template

Each gate below follows this structure:

1. **Principle** — link to `02-security.md` category
2. **Tool(s)** — canonical implementation
3. **Config files** — where settings live
4. **Local setup** — Makefile / pre-commit (when applicable)
5. **CI setup** — job/step YAML example (copy into project workflows)
6. **Thresholds** — what blocks vs what is tracked
7. **Suppressions** — four-field requirement from `02-security.md`

---

## Gate Summary

| Gate | Tool(s) | Stage | Blocks |
|------|---------|-------|--------|
| Python lint + format | Ruff | 1 | Yes |
| Python type check | mypy | 1 | Yes (when enabled) |
| JS/TS lint + format | ESLint + Prettier | 1 | Yes (when frontend) |
| JS/TS type check | tsc --noEmit | 1 | Yes (when frontend) |
| Secret detection | Gitleaks | 1 | Yes |
| Unit tests + coverage | pytest + coverage | 1 | Yes |
| SAST — Python | Bandit | 2 | Yes |
| SAST — JS/TS | eslint-plugin-security | 1–2 | Yes (when frontend) |
| SCA — Python | pip-audit | 2 | Yes |
| SCA — Node | npm audit | 2 | Yes (when frontend) |
| OSS license scan | pip-licenses + license-checker | 2 | Yes |
| IaC scan | Trivy config | 2 | Yes |
| Container image scan | Trivy image | 3 | Yes |
| DAST — release PR | ZAP full | 3 (PR) | Yes |
| DAST — publish | ZAP baseline | 3 | Yes |
| Actions supply chain | zizmor | 2 | Yes |
| Dependency pinning | Renovate or Dependabot | — | Policy |

---

## Stage 1 — Fast Checks

### Python Lint and Format — Ruff

**Principle:** Code quality and consistency (`01-coding-standards.md`).

**Tool:** `ruff` — lint and format in one tool.

**Config files:** `pyproject.toml` → `[tool.ruff]`, `[tool.ruff.lint]`

**Local setup:**

```bash
ruff check .
ruff format --check .
# fix: ruff check --fix . && ruff format .
```

**CI setup:**

```yaml
- name: Install Ruff
  run: pip install ruff==<pinned-version>
- name: Lint
  run: ruff check .
- name: Format check
  run: ruff format --check .
```

**Thresholds:** Any lint or format failure blocks.

**Suppressions:** Rule ignores must be documented in `pyproject.toml` with
reason. Undocumented ignores are not permitted. Do not silently disable
security-relevant rules.

**Notes:** Rule set is project-defined in `pyproject.toml`. Ruff replaces
Flake8, isort, and Black.

---

### Python Type Check — mypy

**Principle:** Static correctness; catches bugs before runtime.

**Tool:** `mypy` in strict mode (when Python is in the project stack).

**Config files:** `pyproject.toml` → `[tool.mypy]`

**Local setup:**

```bash
mypy {source_root}/ --strict
```

**CI setup:**

```yaml
- name: Type check
  run: mypy {source_root}/ --strict
```

**Thresholds:** Any type error blocks when typecheck is enabled in overlay.

**Suppressions:** `# type: ignore` requires inline comment explaining why.

---

### JavaScript / TypeScript Lint and Format

**Principle:** Code quality; partial SAST via security plugin (see Bandit/ESLint).

**Tools:** ESLint with `eslint-plugin-security` + Prettier.

**Config files:** `eslint.config.js`, `.prettierrc`, `package.json`

**Local setup:**

```bash
npm run lint          # eslint --max-warnings=0
npm run format:check  # prettier --check .
```

**CI setup:**

```yaml
- name: Lint
  run: npm run lint
  working-directory: frontend
- name: Format check
  run: npm run format:check
  working-directory: frontend
```

**Thresholds:** `--max-warnings=0`. Warnings block CI.

**Suppressions:** Inline disable comments require explanation.

---

### TypeScript Type Check

**Principle:** Static type safety for frontend code.

**Tool:** `tsc --noEmit`

**Config files:** `tsconfig.json` (`strict: true` required)

**Local setup:** `npx tsc --noEmit`

**CI setup:**

```yaml
- name: TypeScript typecheck
  run: npx tsc --noEmit
  working-directory: frontend
```

**Thresholds:** Any type error blocks.

**Suppressions:** `//@ts-ignore` / `//@ts-expect-error` require inline explanation.

---

### Secret Detection — Gitleaks

**Principle:** [Secret Detection](02-security.md#required-security-scanning-coverage) —
no secrets in source control.

**Tool:** Gitleaks

**Config files:** `.gitleaksignore` (documented false positives only)

**Local setup:** Pre-commit hook (see `05-git-workflow.md`)

**CI setup:**

```yaml
- name: Secret scan
  uses: gitleaks/gitleaks-action@<sha>
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Thresholds:** Any detected secret blocks. **Rotate the secret immediately**
before fixing the pipeline.

**Suppressions:** `.gitleaksignore` entries require inline comments per entry.

---

### Unit Tests and Coverage — pytest

**Principle:** Tests are part of definition of done (`01-coding-standards.md`).

**Tool:** pytest with pytest-cov

**Config files:** `pyproject.toml` → `[tool.pytest.ini_options]`

**Local setup:**

```bash
pytest {test_path}/ -v \
  --cov={source_root} \
  --cov-report=term-missing \
  --cov-fail-under={cov_threshold}
```

**CI setup:**

```yaml
- name: Run unit tests
  run: |
    pytest {test_path}/ -v \
      --cov={source_root} \
      --cov-report=term-missing \
      --cov-report=xml \
      --cov-fail-under={cov_threshold}
```

**Thresholds:** Test failure or coverage below `{cov_threshold}` blocks.
Default greenfield minimum: 80% on `{source_root}`. Overlay may raise threshold
or narrow scope with documented reason.

**Notes:** Stage 1 runs unit tests only — no integration tests.

---

## Stage 2 — Security Scanning

### SAST — Python — Bandit

**Principle:** [SAST](02-security.md#required-security-scanning-coverage)

**Tool:** Bandit

**Config files:** `pyproject.toml` → `[tool.bandit]`

**Local setup:** `bandit -r {source_root}/ -c pyproject.toml`

**CI setup:**

```yaml
- name: Install Bandit
  run: pip install bandit[toml]
- name: SAST scan
  run: bandit -r {source_root}/ -c pyproject.toml
```

**Thresholds:** HIGH and above block. MEDIUM tracked as issues.

**Suppressions:** `# nosec` requires Bandit rule ID and justification.

---

### SAST — JavaScript/TypeScript — ESLint Security Plugin

**Principle:** [SAST](02-security.md#required-security-scanning-coverage)

**Tool:** `eslint-plugin-security` (runs with ESLint in Stage 1)

**Config files:** `eslint.config.js`

```js
import security from 'eslint-plugin-security'
export default [security.configs.recommended]
```

**Thresholds:** Same as ESLint — zero warnings.

**Notes:** Semgrep may be added in overlay for deeper JS/TS SAST.

---

### SCA — Python — pip-audit

**Principle:** [SCA](02-security.md#required-security-scanning-coverage)

**Tool:** pip-audit

**Config files:** `requirements.txt` (fully pinned)

**Local setup:** `pip-audit -r requirements.txt`

**CI setup:**

```yaml
- name: Python dependency audit
  run: pip-audit -r requirements.txt
```

**Thresholds:** HIGH and CRITICAL CVEs block.

**Suppressions:** `--ignore-vuln` with inline workflow comment (CVE, reason, revisit date).

---

### SCA — JavaScript/TypeScript — npm audit

**Principle:** [SCA](02-security.md#required-security-scanning-coverage)

**Tool:** npm audit

**Config files:** `package-lock.json` (committed; use `npm ci` in CI)

**CI setup:**

```yaml
- name: JS dependency audit
  run: npm audit --audit-level=high
  working-directory: frontend
```

**Thresholds:** HIGH and CRITICAL block. MODERATE tracked.

---

### OSS License Scanning

**Principle:** [OSS License Scanning](02-security.md#required-security-scanning-coverage)

**Tools:** `pip-licenses` (Python), `license-checker` (Node)

**CI setup:**

```yaml
- name: Python license scan
  run: |
    pip install pip-licenses
    pip-licenses --fail-on="GPL;AGPL;LGPL" --format=markdown
- name: Node license scan
  run: npx license-checker --failOn "GPL;AGPL;LGPL" --summary
  working-directory: frontend
```

**Thresholds:** Blocked licenses (default: GPL, AGPL, LGPL) fail the gate.

**Suppressions:** New non-permissive licenses require human approval before suppression.

---

### IaC Scanning — Trivy Config Mode

**Principle:** [IaC Scanning](02-security.md#required-security-scanning-coverage)

**Tool:** Trivy `--scanners config`

**Config files:** `.trivyignore`

**Local setup:** `trivy config . --severity HIGH,CRITICAL --exit-code 1`

**CI setup:**

```yaml
- name: IaC scan
  run: |
    trivy config . \
      --severity HIGH,CRITICAL \
      --exit-code 1 \
      --ignorefile .trivyignore
```

**Thresholds:** HIGH and CRITICAL block.

**Suppressions:** `.trivyignore` entries require four-field comment blocks.

**Notes:** Path-filter workflow triggers on Dockerfile, compose, workflows, helm, terraform.

---

### Actions Supply Chain — zizmor

**Principle:** CI/CD pipeline integrity; catches unsafe workflow patterns.

**Tool:** [zizmor](https://github.com/zizmorcore/zizmor)

**Config files:** None required; pin version in Makefile and CI consistently.

**Local setup:** `zizmor --version` (install pinned release); run against `.github/workflows/`

**CI setup:**

```yaml
- name: zizmor workflow audit
  run: |
    curl -fsSL https://github.com/zizmorcore/zizmor/releases/download/v<version>/zizmor-<platform>.tar.gz | tar xz
    ./zizmor --format plain .github/workflows/
```

**Thresholds:** Findings at configured severity block (project defines minimum severity).

**Notes:** Keep `ZIZMOR_VERSION` synced between Makefile and CI job.

---

## Stage 3 — Container and DAST

### Container Image Scanning — Trivy

**Principle:** [Container Image Scanning](02-security.md#required-security-scanning-coverage)

**Tool:** Trivy image mode

**Config files:** `.trivyignore`, Dockerfile (digest-pinned base, non-root user)

**CI setup:**

```yaml
- name: Run Trivy vulnerability scanner
  run: |
    trivy image \
      --severity CRITICAL,HIGH \
      --exit-code 1 \
      --ignorefile .trivyignore \
      <registry>/<image>:scan
```

**Thresholds:** CRITICAL and HIGH block before push to registry.

**Suppressions:** Four-field comment block above each `.trivyignore` entry.

**Base image standards:**

```dockerfile
FROM python:3.14-slim@sha256:<digest>
RUN adduser --disabled-password --gecos '' appuser
USER appuser
```

---

### DAST — ZAP

**Principle:** [DAST](02-security.md#required-security-scanning-coverage)

**Tool:** OWASP ZAP via project scripts against ephemeral container.

| Context | Scan | Script | Trigger |
|---------|------|--------|---------|
| Release PR to `main` | Full (active) | `.github/scripts/zap-full-scan.sh` | PR |
| Push to `main`/`develop` | Baseline (passive) | `.github/scripts/zap-scan.sh` | Publish pipeline |

**CI setup:**

```yaml
- name: ZAP full scan
  run: .github/scripts/zap-full-scan.sh <registry>/<image>:scan ${{ github.workspace }}
- name: Stop container
  if: always()
  run: docker stop <container-name> 2>/dev/null || true
```

**Thresholds:** HIGH and above block.

**Suppressions:** ZAP context/rules file with four-field documentation.

---

## Dependency and Action Pinning

### GitHub Actions SHA Pinning

**Principle:** Supply chain integrity — immutable Action references.

**Policy:** Every `uses:` directive pins a full commit SHA with a version comment:

```yaml
# actions/checkout v4.2.2
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

Tags (`@v4`) and branches (`@main`) are not acceptable.

### Renovate or Dependabot

**Principle:** Automated, reviewable updates for Actions SHAs and container digests.

**Config files:** `renovate.json` or `.github/dependabot.yml` in the **project** repo.

**Setup (Renovate example):**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchManagers": ["dockerfile"],
      "pinDigests": true
    },
    {
      "matchManagers": ["github-actions"],
      "pinDigests": true
    }
  ]
}
```

**Policy:** SHA and digest updates arrive via PR (or direct commit per project workflow).
Never float Action tags in workflow files.

---

## Enforcement Summary

### What Always Blocks

- Any lint or format failure
- Any type error (when typecheck enabled for stack)
- Any secret detected by Gitleaks
- Unit test failure or coverage below `{cov_threshold}`
- Any Bandit HIGH finding
- Any pip-audit or npm audit HIGH/CRITICAL (unless valid suppression)
- Any Trivy CRITICAL/HIGH in image or IaC scan (unless valid suppression)
- Any ZAP HIGH finding
- Any blocked license detected
- zizmor findings at project-configured blocking severity

### Suppression Rules (all tools)

Valid suppressions require all four fields from `02-security.md`:

1. Finding ID
2. Reason
3. Owner
4. Expiry

Agents must not add suppressions without all four fields.

---

## Actions Pinning (reference)

See **Dependency and Action Pinning** above. Use Renovate or Dependabot to
propose SHA updates deliberately — never auto-float tags.
