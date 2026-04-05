# Security Standards

Security is non-negotiable. These rules are not overridable by project-specific
`AGENTS.md` overlays. If a project requirement appears to conflict with a rule
here, stop and escalate to a human — do not work around it.

---

## Secure by Default

Every decision should default to the most restrictive, least privileged, most
auditable option. Security is not added after the fact — it is a constraint on
every design and implementation decision from the start.

- **Deny by default.** Access, permissions, and capabilities start at zero and
  are explicitly granted. Never start permissive and restrict later.
- **Least privilege.** Every component, service account, API key, and IAM role
  gets only the permissions it needs for its specific function — nothing more.
  This applies to agents, pipelines, and application code equally.
- **Minimize attack surface.** Do not expose functionality, ports, endpoints, or
  data that are not required. Every unnecessary exposure is a liability.
- **Assume breach.** Design systems as if an attacker has already compromised one
  component. Lateral movement should be hard. Blast radius should be small.

---

## Secrets Management

**No secret ever touches source control.** This is an absolute rule with no
exceptions, no matter how temporary or how internal the repository.

- **Secrets include**: API keys, tokens, passwords, private keys, certificates,
  connection strings, webhook signing secrets, and any value that grants access
  or proves identity.
- **Secrets are never**: hardcoded in source, committed in `.env` files, embedded
  in Dockerfiles, passed as build args that appear in image layers, or logged.
- **Where secrets live**:
  - Local development: `.env` file excluded by `.gitignore` (`.env.example` with
    dummy values is committed instead)
  - CI/CD: GitHub Actions encrypted secrets — never in workflow YAML as plaintext
  - Production: A dedicated secrets manager (Vault, AWS Secrets Manager, Azure
    Key Vault, etc.) — defined in the project overlay
- **Rotation**: Secrets must be rotatable without a code change or redeployment.
  If rotating a secret requires touching source code, the architecture is wrong.
- **Detection**: A secrets scanner runs on every commit and in CI (see
  `04-quality-gates.md`). If a secret is detected, the pipeline fails. If a
  secret is accidentally committed, treat it as compromised immediately — rotate
  it before doing anything else.
- **Agents**: AI agents must never request, display, log, or transmit secrets.
  If a task requires a secret to be present, it is injected via environment
  variable at runtime. Agents confirm secrets are present, not what they contain.

---

## Input Validation

All input is untrusted until proven otherwise. This includes user input, API
responses, file contents, environment variables, CLI arguments, and inter-service
messages.

- **Validate at the boundary.** Input is validated as soon as it enters the
  system — at the API handler, CLI argument parser, or message consumer. It is
  not passed raw into business logic.
- **Allowlist over blocklist.** Define what is acceptable and reject everything
  else. Do not attempt to enumerate and block known-bad patterns.
- **Validate type, length, format, and range.** A field that expects a UUID
  should reject anything that is not a valid UUID — not just SQL injection
  patterns.
- **Never trust Content-Type.** Validate file contents directly, not based on
  the MIME type or extension the client provides.
- **Reject and log, do not sanitize and continue.** Sanitizing malformed input
  and processing it anyway masks problems and can introduce bypass vulnerabilities.
  Reject invalid input with a clear error and log the attempt with context.
- **Parameterized queries only.** String interpolation into SQL, shell commands,
  LDAP queries, or XML is never acceptable regardless of how the input was
  validated upstream.

---

## Authentication and Authorization

- **Authentication confirms identity. Authorization confirms permission.** Both
  must be checked on every request. Never assume that an authenticated user is
  authorized for a given action.
- **Use established libraries and protocols.** Do not implement custom
  authentication schemes. Use well-audited libraries for OAuth2, JWT, SAML, or
  whatever the project requires (defined in project overlay).
- **JWT handling**:
  - Always validate signature, expiry, issuer, and audience.
  - Never accept `alg: none`.
  - Store tokens in `httpOnly` cookies for web clients, not `localStorage`.
- **Session management**:
  - Invalidate sessions on logout — do not rely on token expiry alone.
  - Rotate session tokens after privilege escalation.
  - Set appropriate expiry for the sensitivity of the resource.
- **Failed authentication**:
  - Return generic error messages. Do not confirm whether a username exists.
  - Implement rate limiting and lockout on authentication endpoints.
  - Log failed attempts with IP, timestamp, and username (not password).

---

## Data Protection

- **Classify data before storing or transmitting it.** Know whether a field is
  public, internal, confidential, or regulated (PII, PHI, PCI). Classification
  is defined in the project overlay.
- **Encrypt in transit.** TLS 1.2 minimum, TLS 1.3 preferred. No plaintext
  protocols for any data transmission. Internal service-to-service traffic is
  not exempt.
- **Encrypt at rest.** Sensitive and regulated data is encrypted at rest.
  Encryption keys are managed separately from the data they protect.
- **PII minimization.** Collect only the personal data that is necessary.
  Do not log PII. Do not include PII in error messages, URLs, or analytics
  events.
- **Data retention.** Data that is no longer needed is deleted. Retention
  periods are defined in the project overlay and enforced programmatically,
  not by manual process.

---

## Dependency and Supply Chain Security

- **All dependencies are scanned** for known vulnerabilities on every CI run
  (see `04-quality-gates.md`). HIGH and CRITICAL findings block the pipeline.
- **No dependency is added without review.** Check: maintenance status, security
  history, license, and transitive dependency footprint.
- **Lockfiles are committed.** `package-lock.json`, `poetry.lock`, `go.sum`, etc.
  are always committed and always used in CI. Do not install from floating ranges.
- **Container base images**:
  - Use minimal base images (distroless, Alpine, slim variants).
  - Pin to a specific digest, not just a tag. Tags are mutable.
  - Run as a non-root user. If a container requires root, document why.
  - Scan every image with Trivy before pushing (see `04-quality-gates.md`).
- **Do not pull untrusted scripts at runtime.** `curl | bash` patterns in
  Dockerfiles or CI pipelines are never acceptable.

---

## Logging and Auditing

- **Log security-relevant events**: authentication attempts, authorization
  failures, privilege escalation, configuration changes, and data exports.
- **Never log secrets, passwords, tokens, or full PII.** Mask or omit sensitive
  fields. Log the fact that a credential was used, not the credential itself.
- **Logs must be tamper-evident.** Ship logs to an external system promptly.
  Local-only logs that can be modified or deleted by a compromised process are
  not sufficient for security events.
- **Include context in every log entry**: timestamp, service name, request ID,
  user or service identity (if known), and the action taken.
- **Structured logging only.** Free-text log lines are not parseable by SIEM
  tools. Use JSON or a structured format defined in the project overlay.

---

## Secure Code Patterns

### Injection Prevention
- SQL: parameterized queries or ORM — never string interpolation
- Shell: avoid shell invocation entirely where possible; use library equivalents;
  never pass user input to shell commands
- HTML: context-aware output encoding; use templating engines that escape by
  default, never build HTML by string concatenation

### Error Handling
- Do not expose stack traces, internal paths, database schemas, or dependency
  versions in error responses to clients.
- Return generic messages to callers; log full detail internally with a
  correlation ID the caller can reference for support.

### Cryptography
- Do not implement custom cryptographic algorithms or protocols.
- Use current recommended algorithms: AES-256-GCM for symmetric encryption,
  RSA-4096 or ECDSA P-256 for asymmetric, SHA-256 or better for hashing.
- Passwords are hashed with a purpose-built algorithm: bcrypt, scrypt, or
  Argon2. Never MD5, SHA-1, or unsalted SHA-256 for passwords.
- Generate cryptographic randomness with a CSPRNG. Never use `Math.random()`
  or equivalent for security-sensitive values.

---

## Required Security Scanning Coverage

The following scanning categories are mandatory for every project. Specific
tooling that satisfies each requirement is defined in `04-quality-gates.md`.
If a category is not covered by the current pipeline, the pipeline is incomplete
and must not be used for production deployments.

| Category | Requirement | When |
|----------|-------------|------|
| **Secret Detection** | Scan every commit and every PR diff for committed secrets, tokens, and credentials | Pre-commit hook + CI on every push |
| **SAST** (Static Application Security Testing) | Scan source code for security vulnerabilities, insecure patterns, and CWE-mapped findings | CI on every push |
| **SCA** (Software Composition Analysis) | Scan all dependencies for known CVEs; block HIGH and CRITICAL unmitigated findings | CI on every push |
| **OSS License Scanning** | Verify all dependency licenses are compatible with the project's license policy | CI on every push |
| **Container Image Scanning** | Scan every built image for OS and package-level CVEs before push to registry | CI after image build, before push |
| **DAST** (Dynamic Application Security Testing) | Run active scanning against a deployed instance for injection, auth, and exposure issues | CI on merge to main, or on-demand against staging |
| **IaC Scanning** | Scan infrastructure-as-code (Terraform, Helm, Dockerfiles) for misconfigurations | CI on every push containing IaC changes |

### Thresholds

These thresholds apply universally. Project overlays may tighten them, never loosen them.

- **CRITICAL findings**: Always block. No exceptions without a documented,
  time-bounded, human-approved exception logged in the PR and tracked issue.
- **HIGH findings**: Block by default. Exceptions require the same process as CRITICAL.
- **MEDIUM findings**: Must be triaged within 30 days. Do not block pipeline
  but are tracked and reported.
- **LOW / INFO findings**: Tracked, reviewed quarterly, do not block pipeline.

### Exception Process

If a finding must be suppressed (e.g., a false positive or an accepted risk):

1. Document the suppression inline with the tool's ignore mechanism.
2. Include a comment with: reason, CVE or finding ID, owner, and expiry date.
3. Call it out explicitly in the PR description.
4. Link to a tracked issue for review at expiry.

Suppressions without all four elements are not valid and will be flagged in review.

---

## Public Repository Safety

All standards documents and project `AGENTS.md` overlays are treated as
public by default. Assume any file in `dev-standards/` or any `AGENTS.md`
will be visible to anyone who can read the repository -- on GitHub, in a
shared codebase, in a code review, or in a cloned copy on any machine.

This means the following must never appear in any of these files under any
circumstances:

- Secrets, passwords, tokens, API keys, or credentials of any kind
- Internal hostnames, IP addresses, or private URLs
- Internal network topology or infrastructure layout details
- Employee names, email addresses, or personal contact information
- Customer data, customer names, or any PII
- Internal system names that reveal security-sensitive architecture
- Vulnerability details or known weaknesses in production systems
- Audit findings, penetration test results, or security assessment output
- Proprietary business logic or trade secrets
- Anything marked confidential, internal-only, or need-to-know

### What Belongs Here Instead

Standards documents and overlays describe *how* to work, not *what exists*.
The right level of detail is process and convention, not implementation
specifics that could aid an attacker or expose private information.

Acceptable in a public `AGENTS.md`:
- The name of an external service the project integrates with (e.g. "Spotify API")
- The name of the environment variable that holds a credential (e.g. `SPOTIFY_CLIENT_ID`)
- The location of a client module (e.g. `src/clients/spotify.py`)
- General data classification (e.g. "this service handles PII")

Not acceptable:
- The actual value of any credential or token
- Internal service URLs or private API endpoints
- Details of security controls that should not be publicly known
- Information about known vulnerabilities in the current codebase

### Agent Responsibility

When generating or updating an `AGENTS.md` overlay, agents must:

1. Never include information from environment variables, config files, or
   runtime context that contains sensitive values
2. Never reference internal infrastructure details learned from scanning
   the codebase or its configuration
3. Flag any section of `AGENTS.template.md` that they cannot complete
   without referencing sensitive information, and leave it as a placeholder
   for the human to complete offline
4. Treat the generated file as if it will be committed to a public GitHub
   repository immediately -- because it will be

If completing a section of the overlay requires writing something that should
not be public, the correct action is to omit it and add a comment:

```markdown
<!-- HUMAN: complete this section offline; do not include sensitive details -->
```

---

## Agent-Specific Security Rules

AI agents operating in this codebase are subject to additional constraints:

- **Agents do not approve their own security exceptions.** Any deviation from
  this document requires explicit human approval in the PR.
- **Agents do not disable security tooling.** Linters, scanners, and hooks are
  not to be bypassed, suppressed, or reconfigured to pass a failing check.
- **Agents do not exfiltrate data.** Agents must not send codebase contents,
  configuration, secrets, or user data to external services beyond those
  explicitly defined in the project overlay.
- **Agents flag and pause on ambiguity.** If a task is unclear and a secure
  interpretation and an insecure interpretation are both plausible, the agent
  stops and asks. It does not pick the path of least resistance.
- **Agents do not store credentials.** Any credential used during a task is
  consumed from the environment at runtime and not written to disk, logs, or
  output files.
