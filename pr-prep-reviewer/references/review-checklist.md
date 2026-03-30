# PR Review Checklist

Comprehensive review criteria organized by category. Work through each category systematically when reviewing a PR diff. Flag every finding with severity: 🔴 Critical, 🟡 Medium, 🔵 Minor.

---

## Category 1 — Correctness

### Logic
- [ ] Does the code correctly implement what the PR description claims?
- [ ] Are all conditional branches handled? (if/else, switch cases, ternary)
- [ ] Are boundary conditions handled? (empty arrays, zero values, max values, page bounds)
- [ ] Are loop termination conditions correct? Could any loop run infinitely?
- [ ] Are there off-by-one errors? (array indices, pagination offsets, date ranges)
- [ ] Are race conditions possible in concurrent/async code?
- [ ] Is mutation of shared state safe? (thread safety, concurrent access)
- [ ] Are type coercions explicit and intentional? (no accidental JS `==` comparisons)
- [ ] Are date/time operations timezone-aware where needed?
- [ ] Are floating-point comparisons using tolerance where appropriate?

### Data Flow
- [ ] Is data correctly transformed at each step?
- [ ] Are all required fields validated before use?
- [ ] Are null/undefined values handled before dereferencing?
- [ ] Is user input sanitized/validated before use?
- [ ] Are database query results validated before accessing properties?

**Flag patterns:**
- `== null` instead of `=== null` (JS)
- `parseInt(x)` without radix
- `new Date(str)` with `-` separator (Safari bug)
- Unguarded array index access: `arr[0].property`
- Missing `await` on async calls

---

## Category 2 — Security

### Secrets & Sensitive Data
- [ ] **No hardcoded secrets:** API keys, tokens, passwords, private keys, connection strings
- [ ] **No sensitive data logged:** passwords, tokens, PII, card numbers
- [ ] **No sensitive data in error messages** returned to clients
- [ ] Are environment variables used instead of hardcoded config values?
- [ ] Are secrets referenced correctly (env vars, secret managers)?

### Input Validation & Injection
- [ ] Is all user input validated before processing?
- [ ] Is SQL built with parameterized queries, not string concatenation?
- [ ] Are HTML outputs escaped to prevent XSS?
- [ ] Are file paths sanitized to prevent path traversal?
- [ ] Are shell commands built with safe escaping (no user input in shell strings)?
- [ ] Are deserialized objects validated before use?

### Authentication & Authorization
- [ ] Are authentication checks present on new/modified endpoints?
- [ ] Are authorization (permission) checks present for sensitive operations?
- [ ] Has any auth check been removed, weakened, or bypassed?
- [ ] Are JWT/session tokens validated correctly?
- [ ] Is rate limiting present on auth endpoints?
- [ ] Are CORS policies correctly configured?

### Cryptography
- [ ] Are deprecated/weak algorithms avoided? (MD5, SHA1 for security, DES, RC4)
- [ ] Are secure random number generators used for tokens/keys?
- [ ] Are passwords hashed with bcrypt/argon2/scrypt (not SHA/MD5)?
- [ ] Are TLS/SSL configurations set to secure minimum versions?

**Flag patterns (🔴 Critical):**
- Any string matching: `sk-`, `api_key`, `secret`, `password`, `token`, `private_key`, `-----BEGIN`
- SQL: `"SELECT * FROM users WHERE id = " + userId`
- HTML: `innerHTML = userInput`
- Commented-out auth checks
- `verify: false` in HTTP/TLS config
- `admin: true` hardcoded in user objects

---

## Category 3 — Readability

- [ ] Are variable/function/class names descriptive and clear?
- [ ] Are there magic numbers without named constants?
- [ ] Is nesting depth reasonable? (max 3-4 levels)
- [ ] Are functions short and single-purpose? (ideally <40 lines)
- [ ] Is complex or non-obvious logic commented?
- [ ] Are boolean conditions readable? (avoid double negatives: `!isNotValid`)
- [ ] Are long chains of operations broken into named intermediate steps?
- [ ] Are error messages clear and helpful?
- [ ] Is the code consistent with surrounding code style?

**Flag patterns (🔵 Minor / 🟡 Medium):**
- `const x = ...` (single-letter variable outside loops)
- `if (!(!isDisabled))` — double negation
- Functions >80 lines
- Nesting depth >4
- Numbers without constants: `if (status === 3)` → should be named
- `// TODO` or `// FIXME` without ticket reference

---

## Category 4 — Maintainability

### Code Structure
- [ ] Is the change well-structured and modular?
- [ ] Is there duplicated logic that should be extracted into a shared function?
- [ ] Are there over-engineered abstractions for simple one-time operations?
- [ ] Is there dead code (unreachable branches, unused variables/imports, commented-out code)?
- [ ] Are dependencies injected rather than hardcoded where appropriate?
- [ ] Does the change follow existing patterns and conventions in the codebase?

### Design Concerns
- [ ] Does a function/class have too many responsibilities?
- [ ] Are public APIs stable and well-defined?
- [ ] Is state managed consistently? (no mixing of local state + global mutations unexpectedly)
- [ ] Are interfaces/types explicit rather than using `any` / `object` / `dict`?

**Flag patterns:**
- Unused imports/variables
- Commented-out blocks of code (delete, don't comment)
- Copy-paste blocks with minor variations (extract to function)
- `any` type in TypeScript without explanation

---

## Category 5 — Test Coverage

### New Code Coverage
- [ ] Are there unit tests for new business logic functions?
- [ ] Are there tests for all public methods/functions added?
- [ ] Are happy path AND unhappy path tested?
- [ ] Are edge cases tested? (empty input, null, max, min, special chars)
- [ ] Are error/exception paths tested?

### Existing Tests
- [ ] Were existing tests updated when behavior changed?
- [ ] Do any existing tests need to be removed (outdated/testing wrong thing)?
- [ ] Are there flaky assertions? (date-dependent, order-dependent, sleep-based)
- [ ] Are mock/stub setups correct and realistic?

### Integration & E2E
- [ ] If the change affects an API, are integration tests present?
- [ ] If the change affects a user flow, is E2E coverage considered?
- [ ] Are database interactions tested with realistic data?

**Common gaps to flag:**
- New utility function with no tests
- Error handler with no test for the error case
- Auth guard with no test for unauthorized access
- Pagination logic with no test for empty page, last page, out-of-bounds

---

## Category 6 — Documentation

### Code Documentation
- [ ] Are complex functions/classes documented with purpose and parameters?
- [ ] Are non-obvious decisions explained in inline comments?
- [ ] Are deprecated functions/APIs marked with `@deprecated`?
- [ ] Are TODO/FIXME comments linked to issues?

### External Documentation
- [ ] If a new API endpoint was added → API docs updated?
- [ ] If behavior changed → README updated where relevant?
- [ ] If new config/env vars added → documented in README or .env.example?
- [ ] If CLI commands changed → usage docs updated?
- [ ] If database schema changed → migration notes present?
- [ ] If public API (SDK/library) changed → changelog updated?

---

## Category 7 — Backward Compatibility

### API Compatibility
- [ ] Were any existing API endpoints removed or renamed?
- [ ] Were any required request/response fields removed?
- [ ] Were any field types or formats changed?
- [ ] Were any API behaviors changed in a breaking way?
- [ ] If breaking: is versioning or deprecation strategy in place?

### Data Compatibility
- [ ] Were any database schema changes made? Are migrations included?
- [ ] Are migrations reversible (down migration provided)?
- [ ] Will the migration run safely on production data size?
- [ ] Is there a zero-downtime deployment path for schema changes?
- [ ] Were any serialized data formats changed (JSON keys, protobuf fields)?

### Code Compatibility
- [ ] Is the minimum runtime/language version still satisfied?
- [ ] Were any dependencies removed that other code still uses?
- [ ] Were any environment variables renamed or removed?

---

## Category 8 — Performance

- [ ] Are there N+1 query patterns? (database query inside a loop)
- [ ] Are expensive operations cached where appropriate?
- [ ] Are database queries indexed for the access patterns used?
- [ ] Are large datasets paginated or streamed instead of loaded all at once?
- [ ] Are synchronous blocking operations avoidable in async contexts?
- [ ] Are there memory leak risks? (event listeners not removed, large closures retained)
- [ ] Are compute-heavy operations debounced or throttled in UI code?
- [ ] Are large response payloads compressed or paginated?

---

## Category 9 — Error Handling

- [ ] Are all async operations wrapped in proper error handling?
- [ ] Are error messages safe for end users? (no stack traces or internals exposed)
- [ ] Are errors logged with enough context for debugging?
- [ ] Do failed operations leave the system in a consistent state?
- [ ] Are retries implemented where appropriate for transient failures?
- [ ] Are timeout values set on external calls?
- [ ] Are network/IO errors handled gracefully with user-friendly messages?
- [ ] Are 3rd party service failures handled without crashing the app?

---

## Category 10 — Dependencies

- [ ] Is the new dependency well-maintained and actively used?
- [ ] Does the dependency have known security vulnerabilities? (check: `npm audit`, `safety`, `snyk`)
- [ ] Is the dependency license compatible with the project?
- [ ] Is the dependency scope correct? (`devDependency` vs `dependency`)
- [ ] Does the dependency significantly increase bundle size?
- [ ] Could the functionality be achieved without a new dependency?

---

## Severity Classification Guide

| Severity | When to use | Must fix before merge? |
|----------|-------------|----------------------|
| 🔴 Critical | Security vulnerability, data loss risk, production crash risk, auth bypass | Yes |
| 🟡 Medium | Missing tests for critical logic, backward compat break without docs, significant performance issue | Should fix |
| 🔵 Minor | Style, naming, small readability issues, optional improvements | Nice to have |
| ❓ Question | Intent unclear, design decision needs explanation | Needs author clarification |

---

## Merge Readiness Decision

| Condition | Verdict |
|-----------|---------|
| No critical issues, tests present, description clear | READY TO MERGE |
| Medium issues found, no blockers | APPROVE WITH SUGGESTIONS |
| Critical issues found (security, correctness, missing tests) | REQUEST CHANGES |
| Intent/design unclear, missing context, needs discussion | NEEDS DISCUSSION |
| Secrets in code, auth bypass, production risk | BLOCKED — DO NOT MERGE |
