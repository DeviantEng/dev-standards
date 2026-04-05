# Quality Gates

This document defines the specific tooling, commands, thresholds, and failure
conditions that fulfill the scanning requirements defined in `02-security.md`
and the pipeline stages defined in `03-cicd.md`. Every gate listed here must
pass before a change is considered shippable.

Tooling choices are recorded here with rationale. If a tool is replaced, update
this document and the corresponding workflow files in the same PR.

---

## Gate Summary

| Gate | Tool(s) | Blocks Pipeline | Workflow | Stage |
|------|---------|-----------------|----------|-------|
| Python lint + format | Ruff | Yes | pr-checks, publish | 1 |
| Python type check | mypy | Yes | pr-checks | 1 |
| JS/TS lint + format | ESLint + Prettier | Yes | pr-checks | 1 |
| JS/TS type check | tsc --noEmit | Yes | pr-checks | 1 |
| Secret detection | Gitleaks | Yes | pr-checks | 1 |
| Unit tests + coverage | pytest + coverage | Yes | pr-checks, publish | 1 |
| SAST — Python | Bandit | Yes | pr-checks | 2 |
| SAST — JS/TS | ESLint security plugin | Yes | pr-checks | 2 |
| SCA — Python | pip-audit | Yes | pr-checks | 2 |
| SCA — JS/TS | npm audit | Yes | pr-checks | 2 |
| OSS license scan | pip-licenses + license-checker | Yes | pr-checks | 2 |
| IaC scan | Trivy (config mode) | Yes | pr-checks | 2 |
| Container image scan | Trivy (image mode) | Yes | publish | 3 |
| DAST — PR to main | ZAP full scan | Yes | pr-checks | 3 (PR) |
| DAST — push to main/develop | ZAP baseline scan | Yes | publish | 3 |

---

## Stage 1 — Fast Checks

### Python Lint and Format — Ruff

**Tool**: `ruff` — covers lint and format in a single fast tool.

```yaml
- name: Install Ruff
  run: pip install ruff==<pinned-version>

- name: Lint
  run: ruff check .

- name: Format check
  run: ruff format --check .
```

- Configuration lives in `pyproject.toml` under `[tool.ruff]`.
- Zero warnings policy: `--select ALL` with explicit ignores documented in
  `pyproject.toml`. Undocumented ignores are not permitted.
- Ruff replaces Flake8, isort, and Black — do not add those separately.

### Python Type Check — mypy

**Tool**: `mypy` in strict mode.

```yaml
- name: Type check
  run: mypy src/ --strict
```

- Configuration lives in `pyproject.toml` under `[tool.mypy]`.
- `# type: ignore` suppressions require an inline comment explaining why.
  Bare `# type: ignore` with no explanation is treated as a lint violation.

### JavaScript / TypeScript Lint and Format — ESLint + Prettier

**Tools**: ESLint with `eslint-plugin-security` + Prettier.

```yaml
- name: Lint
  run: npm run lint          # eslint --max-warnings=0

- name: Format check
  run: npm run format:check  # prettier --check .
```

- `--max-warnings=0` is required — warnings are not acceptable in CI.
- ESLint config must include `eslint-plugin-security`. This doubles as
  partial SAST coverage for the frontend (see Stage 2).
- Prettier config lives in `.prettierrc`. No inline format disable comments
  without an explanation.

### TypeScript Type Check

```yaml
- name: TypeScript typecheck
  run: npx tsc --noEmit
```

- `strict: true` is required in `tsconfig.json`. No exceptions.
- `//@ts-ignore` and `//@ts-expect-error` require an inline explanation comment.

### Secret Detection — Gitleaks

**Tool**: Gitleaks — scans the git diff on every push and PR for committed
secrets, tokens, and credentials.

```yaml
- name: Secret scan
  uses: gitleaks/gitleaks-action@<sha>
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- Runs on every push to every branch and every PR.
- **If Gitleaks fires, treat the secret as compromised immediately.** Rotate
  before addressing the pipeline failure.
- Allowed suppressions: `.gitleaksignore` file with inline comments explaining
  each entry. False positives must be documented — empty suppressions are invalid.
- Pre-commit hook (see `05-git-workflow.md`) runs Gitleaks locally before push
  as an early warning. CI is the enforcement layer.

### Unit Tests and Coverage — pytest

**Tool**: pytest with pytest-cov.

```yaml
- name: Run unit tests
  run: |
    pytest tests/unit/ -v \
      --cov=src \
      --cov-report=term-missing \
      --cov-report=xml \
      --cov-fail-under=80
```

- `--cov-fail-under=80` is the shared minimum. Project overlay may raise this.
- Coverage report is uploaded as a CI artifact for review.
- `tests/unit/` only in Stage 1 — no integration tests here.
- Test run must complete within the timeout defined in `03-cicd.md` (10 minutes).
- A passing test suite that drops below the coverage threshold **fails the gate**.

---

## Stage 2 — Security Scanning

### SAST — Python — Bandit

**Tool**: Bandit — static analysis for common Python security issues.

```yaml
- name: Install Bandit
  run: pip install bandit[toml]

- name: SAST scan
  run: bandit -r src/ -c pyproject.toml
```

- Configuration in `pyproject.toml` under `[tool.bandit]`.
- Severity threshold: HIGH and above block the pipeline. MEDIUM findings are
  reported but do not block (they are tracked as issues).
- `# nosec` suppressions require an inline comment with the Bandit rule ID
  and justification. Bare `# nosec` is not valid.

### SAST — JavaScript/TypeScript — ESLint Security Plugin

ESLint with `eslint-plugin-security` running in Stage 1 provides SAST coverage
for the frontend. No separate tool is required if the plugin is configured with
the full recommended ruleset:

```js
// eslint.config.js
import security from 'eslint-plugin-security'
export default [security.configs.recommended]
```

For deeper JS/TS SAST coverage on security-sensitive projects, Semgrep may be
added as a project overlay tool.

### SCA — Python — pip-audit

**Tool**: pip-audit — scans Python dependencies against OSV and PyPI Advisory DB.

```yaml
- name: Install pip-audit
  run: pip install pip-audit

- name: Python dependency audit
  run: pip-audit -r requirements.txt
```

- HIGH and CRITICAL CVEs block the pipeline.
- Suppressions use `--ignore-vuln <CVE-ID>` and **must** be accompanied by an
  inline comment in the workflow file explaining: the CVE, why it is suppressed,
  and when it should be revisited. This matches the exception process in
  `02-security.md`.

  ```yaml
  # CVE-2026-XXXX: No fix available upstream as of <date>. 
  # Transitive via <package>. Revisit after <package> releases ><version>.
  run: pip-audit -r requirements.txt --ignore-vuln CVE-2026-XXXX
  ```

- `requirements.txt` must be fully pinned. Floating ranges cause audit drift.

### SCA — JavaScript/TypeScript — npm audit

**Tool**: npm audit (built-in).

```yaml
- name: JS dependency audit
  run: npm audit --audit-level=high
  working-directory: frontend
```

- `--audit-level=high` blocks on HIGH and CRITICAL. MODERATE and below are
  reported, not blocking.
- Lockfile (`package-lock.json`) must be committed and used. `npm ci` is used
  in all CI jobs, never `npm install`.

### OSS License Scanning

**Tools**: `pip-licenses` (Python) + `license-checker` (Node).

```yaml
# Python
- name: Python license scan
  run: |
    pip install pip-licenses
    pip-licenses --fail-on="GPL;AGPL;LGPL" --format=markdown

# Node
- name: Node license scan
  run: |
    npx license-checker --failOn "GPL;AGPL;LGPL" --summary
  working-directory: frontend
```

- Blocked licenses by default: GPL, AGPL, LGPL (copyleft licenses incompatible
  with typical proprietary or MIT-licensed projects).
- Project overlay defines the allowed license policy if the defaults do not apply.
- New dependencies with non-permissive licenses require explicit human approval
  before the suppression is added.

### IaC Scanning — Trivy Config Mode

**Tool**: Trivy in `--scanners config` mode — catches Dockerfile, Helm, and
Terraform misconfigurations.

```yaml
- name: IaC scan
  run: |
    trivy config . \
      --severity HIGH,CRITICAL \
      --exit-code 1 \
      --ignorefile .trivyignore
```

- Runs on every push containing changes to `Dockerfile`, `docker-compose*.yml`,
  `.github/workflows/`, `helm/`, or `terraform/` paths. Use path filters in
  the workflow trigger to avoid unnecessary runs.
- Findings are reported in the job summary alongside the image scan results.

---

## Stage 3 — Container Image Scanning — Trivy

**Tool**: Trivy in image mode. This is the primary container gate.

```yaml
- name: Run Trivy vulnerability scanner
  run: |
    # Surface ignored CVEs in job summary for visibility
    if [ -f .trivyignore ]; then
      echo "### Ignored CVEs (review at expiry dates)" >> $GITHUB_STEP_SUMMARY
      echo '```' >> $GITHUB_STEP_SUMMARY
      cat .trivyignore >> $GITHUB_STEP_SUMMARY
      echo '```' >> $GITHUB_STEP_SUMMARY
    fi

    echo "## Trivy Vulnerability Scan" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY

    # Capture output and exit code separately so tee doesn't mask failures
    set +e
    output=$(trivy image \
      --severity CRITICAL,HIGH \
      --exit-code 1 \
      --format table \
      --ignorefile .trivyignore \
      <registry>/<image>:scan 2>&1)
    exitcode=$?
    set -e

    echo "$output" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
    exit $exitcode
```

- **Image is built first, scanned before push.** A failing scan prevents the
  image from reaching the registry.
- `.trivyignore` entries require a comment block above each entry:
  ```
  # CVE-2026-XXXX
  # Reason: No fix available in upstream <package> as of <date>.
  # Revisit: When <package> releases ><version>.
  # Owner: <team or person>
  CVE-2026-XXXX
  ```
- Entries without all four comment fields are not valid and must be corrected.
- Trivy Action is pinned to a full commit SHA — not a tag. Update the SHA
  deliberately with a PR, not automatically.
- The full Trivy output is written to the GitHub Actions job summary for
  visibility without requiring log inspection.

### Base Image Standards

- Use minimal base images: `distroless`, `alpine`, or `-slim` variants.
- Pin base images to a digest in the Dockerfile, not a tag:
  ```dockerfile
  # ✅
  FROM python:3.14-slim@sha256:<digest>
  # ❌
  FROM python:3.14-slim
  ```
- Run as a non-root user. Every Dockerfile must include:
  ```dockerfile
  RUN adduser --disabled-password --gecos '' appuser
  USER appuser
  ```
- Do not install unnecessary packages. Every layer adds attack surface.

---

## DAST — ZAP

**Tool**: OWASP ZAP, run via custom scripts against an ephemeral containerized
instance of the application. No persistent staging environment is required —
the container is started within the pipeline run and torn down after scanning.

Two scan tiers are used depending on pipeline context:

| Context | Workflow | Scan type | Script | Trigger |
|---------|----------|-----------|--------|---------|
| PR to `main` | `pr-checks.yml` | Full scan (active) | `.github/scripts/zap-full-scan.sh` | PR opened or updated |
| Push to `main` or `develop` | `publish.yml` | Baseline scan (passive) | `.github/scripts/zap-scan.sh` | Post-build, pre-push |

### How It Works

1. The Docker image is built with a `:scan` tag (single-arch, not pushed).
2. The image is started as a container (`cmdarr-zap-target` or equivalent).
3. ZAP scans the running container on its exposed port.
4. The container is stopped in an `if: always()` step regardless of scan outcome.
5. On the publish pipeline, the image is only pushed if the ZAP scan passes
   (`if: success()` guard on the build-and-push step).

```yaml
- name: ZAP full scan        # pr-checks.yml
  run: .github/scripts/zap-full-scan.sh <registry>/<image>:scan ${{ github.workspace }}

- name: ZAP baseline scan    # publish.yml
  run: .github/scripts/zap-scan.sh <registry>/<image>:scan ${{ github.workspace }}

- name: Stop container
  if: always()
  run: docker stop <container-name> 2>/dev/null || true
```

### Scan Tiers

- **Full scan (PR)**: Active scanning enabled. ZAP attempts to interact with
  the application to discover injection, authentication, and exposure issues.
  Longer duration, broader coverage. Blocks PR merge on HIGH findings.
- **Baseline scan (main/develop)**: Passive scanning only. ZAP spiders the
  application and reports findings without active probing. Faster. Confirms
  no regressions introduced after merge. Blocks image push on HIGH findings.

### Thresholds and Suppressions

- ZAP findings at HIGH or above block the pipeline in both scan tiers.
- False positives are suppressed via a ZAP context or rules file in `.github/zap/`.
- Suppressions follow the four-field documentation requirement (finding ID,
  reason, owner, expiry) consistent with Trivy and pip-audit suppression policy.

---

## Enforcement Summary

### What Always Blocks

- Any lint or format failure
- Any type error
- Any secret detected by Gitleaks
- Unit test failure
- Coverage below project threshold (minimum 80%)
- Any Bandit HIGH finding
- Any pip-audit or npm audit HIGH/CRITICAL CVE (unless suppressed with valid documentation)
- Any Trivy CRITICAL or HIGH finding in image or IaC scan (unless suppressed with valid documentation)
- Any ZAP HIGH finding
- Any blocked license detected

### What Never Blocks But Is Always Tracked

- Bandit MEDIUM findings → GitHub Issue, triaged within 30 days
- npm audit MODERATE findings → GitHub Issue, triaged within 30 days
- Trivy MEDIUM findings → GitHub Issue, triaged within 30 days
- ZAP MEDIUM findings → GitHub Issue, triaged within 30 days

### Suppression Rules (applies to all tools)

A suppression is only valid when it includes all of the following:

1. **Finding ID** — CVE number, rule ID, or ZAP alert name
2. **Reason** — why this is suppressed (false positive, no fix available, accepted risk)
3. **Owner** — team or person responsible for review
4. **Expiry** — date or milestone at which the suppression must be re-evaluated

Suppressions missing any field are flagged in PR review and must be corrected
before merge. Agents must not add suppressions without all four fields.

---

## Actions Pinning

All GitHub Actions used in pipelines must be pinned to a full commit SHA.
This applies to every `uses:` directive without exception.

```yaml
# ✅ Correct
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

# ❌ Not acceptable
uses: actions/checkout@v4
uses: actions/checkout@main
```

Maintain a comment above each pinned action with the human-readable version
for maintainability:

```yaml
# actions/checkout v4.2.2
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

Use a tool such as `pin-github-action` or Dependabot to manage SHA updates
deliberately via PR rather than manually.
