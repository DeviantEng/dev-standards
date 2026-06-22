# Coding Standards

These standards apply to all languages and projects unless explicitly overridden
in a project-specific `AGENTS.md`. Language-specific conventions (formatting
rules, import ordering, type systems) are defined in the project overlay — the
principles here are universal.

---

## Modularity and Single Responsibility

Every function, class, and module should do one thing and do it well.

- **Functions**: If a function needs more than one paragraph to describe what it
  does, it should be split. If it exceeds ~40 lines (excluding comments), treat
  that as a signal to refactor — not a hard rule, but a prompt to question scope.
- **Modules/Files**: Group by feature or domain, not by type. Prefer
  `auth/login.py` over `handlers/auth.py` sitting next to `handlers/payments.py`.
- **No god objects or god modules**: A file that is imported by everything and
  imports everything is an architectural problem. Flag it.
- **Interfaces before implementation**: Define the contract (function signature,
  API shape, interface/protocol) before writing the body. This forces clarity
  about inputs, outputs, and side effects upfront.

---

## Reuse and DRY

**No identical block of logic should exist in more than one place in the
codebase.** This is a hard rule. If duplication is detected during review or by
an agent, it must be resolved — not noted and deferred.

- **Before writing new code**, search the codebase for existing implementations.
  Agents must perform this search explicitly, not assume nothing exists.
- **The threshold for extraction is two occurrences.** If identical or near-identical
  logic appears in two places, it is extracted immediately into a shared module.
  Do not wait for a third.
- **Near-identical counts.** Code that differs only by variable names or minor
  constants is duplication. Extract it with parameters.
- **Shared utilities** live in a clearly named directory at the appropriate scope:

  | Scope | Location |
  |-------|----------|
  | Used by one module | Inline helper in that module |
  | Used across modules in one project | `src/utils/` or `src/lib/` |
  | Used across multiple projects | Dedicated shared library repo |

- **Do not reuse code by copy-paste across projects.** If a pattern is needed in
  two projects, it belongs in a shared library with its own versioning.
- **Prefer composition over inheritance.** Favor small, combinable functions over
  deep class hierarchies.
- **If duplication exists in code you are touching**, you are responsible for
  resolving it, even if you did not introduce it. Flag it in the PR if the fix
  is out of scope for the current task, but do not silently leave it.

---

## Naming

Names are documentation. They should be specific, honest, and consistent.

- **Be descriptive**: `get_user_by_email()` not `get_user()` or `fetch()`.
- **Be honest**: A function named `validate_input()` should not also write to
  a database. Name reflects behavior.
- **Avoid abbreviations** unless they are universally understood in the domain
  (e.g., `id`, `url`, `http`). `usr`, `cfg`, `mgr` are not acceptable.
- **Booleans**: Prefix with `is_`, `has_`, `can_`, or `should_`. Never name a
  boolean `flag`, `status`, or `data`.
- **Collections**: Use plural nouns (`users`, `error_messages`).
- **Constants**: `UPPER_SNAKE_CASE` universally, regardless of language.
- **Consistency**: If the codebase uses `user_id`, do not introduce `userId` or
  `uid`. Match existing conventions in the project overlay.

---

## Error Handling

Errors are first-class citizens, not afterthoughts.

- **Never silently swallow errors.** An empty `except`, `catch`, or `catch (e) {}`
  block is always wrong. At minimum, log the error with context.
- **Fail loudly in development, gracefully in production.** Surface errors with
  full context during development. In production, return safe error responses to
  callers while logging the full detail internally.
- **Typed errors over generic ones.** Define specific error types/classes for
  domain errors. Catching `Exception` or the equivalent base class is a last
  resort only at application boundaries.
- **Error messages must include context.** `"Database error"` is useless.
  `"Failed to fetch user id=42 from users table: connection timeout"` is useful.
- **Every external call is a potential failure.** Network calls, disk I/O,
  third-party APIs, and subprocess calls must always have explicit error handling.
- **Do not use exceptions for control flow.** Exceptions are for unexpected
  conditions, not for branching logic like "user not found."

---

## Tech Debt

Tech debt is sometimes unavoidable. It must always be visible and tracked.

- **Document it inline** at the point it is introduced:
  ```
  # TODO(debt): This bypasses the rate limiter due to time constraints.
  # Tracked: <issue URL>
  # Owner: <name or team>
  # Acceptable until: <date or milestone>
  ```
- **All four fields are required.** A `TODO` without a tracker link and an
  owner is not a debt marker — it is abandoned code.
- **No debt is introduced silently.** If a PR contains a `TODO(debt):`, it must
  be called out explicitly in the PR description.
- **Debt is not inherited.** Refactoring a file does not mean carrying forward
  its existing undocumented debt. Surface it, document it, or resolve it.
- **Scheduled review.** Debt items older than 90 days without movement should
  be escalated or resolved in the next planning cycle.

---

## Code Clarity

- **Comments explain *why*, not *what*.** The code shows what is happening.
  Comments explain intent, tradeoffs, or non-obvious decisions.
- **Delete dead code.** Do not comment out old code and leave it. Source control
  is the history — use it.
- **No magic numbers or strings.** Literals that carry meaning belong in named
  constants. `MAX_RETRY_ATTEMPTS = 3` not `for i in range(3)`.
- **Keep nesting shallow.** Prefer early returns and guard clauses over deeply
  nested conditionals. More than 3 levels of nesting is a refactor signal.
- **Limit side effects.** Functions that return a value should generally not also
  mutate global state, write to disk, or make network calls. Separate pure logic
  from side-effectful operations.

---

## Dependencies

- **Prefer standard library over third-party** when the functionality is
  comparable. Every dependency is a future maintenance burden and attack surface.
- **Pin versions.** Floating dependencies (`>=1.0`) are not acceptable in
  production code. Use exact pins or lockfiles.
- **Review before adding.** Before adding a new dependency, confirm:
  - Is it actively maintained?
  - Does it have a reasonable security track record?
  - Is its license compatible with the project?
- **Remove unused dependencies.** Regularly audit and prune. Dead dependencies
  still appear in vulnerability scans.

---

## Testing

Tests are not a follow-up task. They are part of the definition of done.

### When to Write Tests

| Situation | Required test type |
|-----------|--------------------|
| New function or module | Unit test, same PR |
| New API endpoint or CLI command | Integration test, same PR |
| Bug fix | Regression test that would have caught the bug, same PR |
| Refactor of existing code | Existing tests must pass; add tests for any uncovered paths found |
| External boundary (DB, API, queue, filesystem) | Integration test with controlled environment |

**Agents must write tests in the same commit as the code they cover.** Tests
in a separate follow-up PR are not acceptable unless explicitly approved by a
human reviewer with a documented reason.

### Test Structure

- **Arrange-Act-Assert** for all unit tests. One logical outcome per test.
- **Test names describe the scenario**, not the function:
  - ✅ `test_login_returns_401_when_password_is_wrong`
  - ❌ `test_login_2`
- **Cover the unhappy path.** Invalid inputs, boundary values, empty collections,
  network failures, and permission errors must all be tested. Happy-path-only
  suites will not pass the quality gate.
- **Test behavior, not implementation.** Tests must survive internal refactoring.
  If renaming a private helper breaks a test, the test is wrong.

### Test Co-location

Test file location is defined in the project overlay. One of two conventions
must be chosen and applied consistently — do not mix:

- **Co-located**: `src/auth/login.py` → `src/auth/test_login.py`
- **Mirrored**: `src/auth/login.py` → `tests/auth/test_login.py`

### Mocking

- **Mock at the boundary**, not inside the implementation. Mock the HTTP client,
  not the function three layers deep that calls it.
- **Do not mock what you own.** Internal modules should be tested directly, not
  mocked away.
- **Mocks must be realistic.** A mock that can never fail does not test error
  handling. Mocks should exercise both success and failure paths.
- **No live external calls in unit tests.** Any test that calls a real database,
  real API, or real filesystem is an integration test and must be labeled and
  isolated accordingly.

### Coverage

- The shared minimum is **80% line coverage**. Project overlays may raise this.
- Coverage is a floor, not a goal. 80% with meaningful assertions beats 100%
  with assertions that never fail.
- Coverage reports are generated in CI. Tests that pass but drop coverage below
  the threshold fail the quality gate (see `04-quality-gates.md`).

---

## Project Structure

Every project declares a **layout profile** in its root `AGENTS.md` overlay.
The profile defines where source code and tests live. Deviations from the
profile skeleton are documented in the overlay — not as a full directory tree.

### Layout Profiles

#### `src-layout` (greenfield default)

Application source under `src/`. Tests under `tests/unit/` and
`tests/integration/`.

```
project-root/
├── AGENTS.md                  # Project entry point — generated at project root
├── dev-standards/             # Git submodule — do not edit
├── README.md
├── src/
│   ├── clients/
│   ├── models/
│   ├── services/
│   ├── utils/
│   └── config/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── scripts/
├── docs/
└── .github/workflows/
```

**Overlay paths:** `{source_root}` = `src/`, `{test_path}` = `tests/unit/`

#### `app-root-layout` (Python/FastAPI common)

Application packages at repo root. Tests mirrored as flat `tests/test_*.py`.

```
project-root/
├── AGENTS.md
├── dev-standards/
├── app/                       # or domain-specific top-level packages
├── commands/
├── clients/
├── tests/
│   └── test_*.py
├── docs/
└── .github/workflows/
```

**Overlay paths:** `{source_root}` = `app/` (or declared package roots),
`{test_path}` = `tests/`

Declare the profile and any deviations in the project `AGENTS.md`. Quality
gate commands in `04-quality-gates.md` use `{source_root}` and `{test_path}`
placeholders — substitute the values from your overlay.

### Directory Rules (apply within declared profile)

- **`clients/`**: One file per external dependency. A client wraps all
  interaction with one external system. No business logic lives here — only
  the mechanics of the call, retry, and error translation.
- **`services/`**: Business logic only. Services call clients and models.
  Services do not know about HTTP, SQL syntax, or queue protocols — that is
  the client's job.
- **`utils/`**: Pure functions only. No imports from `clients/` or `services/`.
  If a utility needs to make a network call, it is not a utility — it is a
  service or client.
- **`config/`**: All configuration is loaded and validated here at startup.
  No other module reads environment variables directly. Config is injected as
  a dependency, never imported as a global.
- **`docs/`**: Architecture Decision Records (ADRs) for any non-obvious
  technical decision. Format: `docs/adr/NNNN-short-title.md`.

### What Does Not Have a Home

If a new file does not fit cleanly into the above structure, that is a signal
to discuss the architecture before writing the code — not to create an
ad-hoc directory.

---

## Language-Specific Rules

Language-specific conventions (linter config, formatter, type strictness, import
order, async patterns) are defined in the consuming project's `AGENTS.md` overlay.
When starting a new project, copy `dev-standards/AGENTS.template.md` to the
project root as `AGENTS.md` and use an AI agent to fill it in. See
`dev-standards/00-overview.md` for setup instructions.
