# Code Review Standards
# Spring Boot + Python + Flutter + RAG + Orchestration

Standards Version: v1.0

---

## I. REVIEW PHILOSOPHY

- Code review is a quality gate, not a style debate. Focus on correctness, security, performance, and maintainability.
- Every finding must be actionable: state what is wrong, why it matters, and what the fix should be.
- Severity must be explicit: P0 (block), P1 (must fix before merge), P2 (should fix), P3 (suggestion).
- Reviews must be objective and tied to standards — not personal preference.

---

## II. UNIVERSAL REVIEW CHECKLIST (ALL STACKS)

### Correctness
- [ ] Does the code implement the intended behavior as specified in the design doc / ticket?
- [ ] Are all edge cases handled (null inputs, empty collections, concurrent access, network failure)?
- [ ] Are error conditions propagated correctly (not swallowed silently)?
- [ ] Are all async operations properly awaited?

### Security
- [ ] No secrets, tokens, or credentials in code or config files.
- [ ] All user inputs validated before use (SQL injection, command injection, path traversal).
- [ ] PII is masked in logs and never serialized to unencrypted storage.
- [ ] Authentication and authorization checks are present on all protected endpoints.
- [ ] Dependency versions pinned — no unpinned `latest` or `*` versions.
- [ ] No unsafe deserialization of external input.

### Performance
- [ ] No N+1 query patterns (check for loops calling DB or APIs individually).
- [ ] Unbounded queries include pagination or a hard limit.
- [ ] No synchronous blocking I/O on async threads.
- [ ] Caches used where appropriate; TTL and invalidation policy defined.
- [ ] Resource cleanup guaranteed (connections, file handles, streams closed in finally/context manager).

### Maintainability
- [ ] Single Responsibility: each class/function does one thing.
- [ ] No magic numbers or hardcoded strings — constants or config values.
- [ ] No dead code, commented-out blocks, or TODO items older than the current sprint.
- [ ] Public methods and complex logic have inline documentation.
- [ ] Cyclomatic complexity ≤ 10 per function.

### Testing
- [ ] New code has corresponding unit tests covering happy path and failure paths.
- [ ] Tests are deterministic — no random data without seeding, no time-dependent assertions without mocking.
- [ ] Test coverage does not drop below threshold.
- [ ] Tests assert behavior, not implementation details.

---

## III. SPRING BOOT — Specific Review Rules

### API Layer
- [ ] Controller methods are thin — no business logic in controllers.
- [ ] All request bodies annotated with `@Valid`.
- [ ] All endpoints versioned under `/api/v{n}/`.
- [ ] Write endpoints include `Idempotency-Key` handling.
- [ ] HTTP status codes are semantically correct (201 for create, 204 for delete, 409 for conflict).
- [ ] Error responses follow the standard error model (no raw exception messages exposed).

### Service / Domain Layer
- [ ] No direct repository calls from controllers.
- [ ] `@Transactional` on service methods with write operations.
- [ ] Transactions are not too broad — don't include external service calls inside transaction boundary.
- [ ] Domain logic lives in domain objects/services — not in application service as procedural code.

### Data Access
- [ ] No `findAll()` without pagination.
- [ ] Queries use indexed columns in WHERE clauses.
- [ ] `@Version` on entities used for concurrent updates.
- [ ] New Flyway migrations are additive-only; no destructive changes without rollback script.
- [ ] JPA `FetchType.LAZY` used for associations — no `EAGER` without explicit justification.

### Security (Spring-Specific)
- [ ] `@PreAuthorize` on service methods for role-gated operations.
- [ ] No `permitAll()` on write endpoints.
- [ ] CORS config does not use wildcard origin in production profile.
- [ ] Secrets injected from Secrets Manager — not from `application.properties` or env vars in plain text.

---

## IV. PYTHON AI SERVICES — Specific Review Rules

### FastAPI / Service Layer
- [ ] All routes are `async def`. No synchronous routes for I/O-bound operations.
- [ ] Pydantic models used for all request/response shapes. No raw `dict` passed through layers.
- [ ] `Depends()` used for dependency injection — no manual instantiation inside route handlers.
- [ ] Proper HTTP status codes returned (422 for validation, 404 for not found, etc.).

### RAG-Specific
- [ ] Embedding model name and version explicitly set in config — not defaulted.
- [ ] Chunking function tested with edge cases (empty documents, very short docs, multilingual content).
- [ ] Vector store `upsert` uses content hash to detect and skip duplicate chunks.
- [ ] Retrieval top-K is configurable — not hardcoded.
- [ ] Prompt assembled with structured delimiters — not raw string concatenation of chunks.
- [ ] LLM output validated/parsed before returning to caller — never raw pass-through.
- [ ] Token budget checked before LLM call — truncation strategy defined for overflow.
- [ ] All LLM calls logged with `model_id`, `prompt_tokens`, `completion_tokens`, `latency_ms`.

### Orchestration-Specific
- [ ] Every tool has `name`, `description`, `input_schema`, `output_schema`, `timeout_seconds`.
- [ ] Tool inputs validated with Pydantic before execution.
- [ ] Agent loop has explicit `max_turns` guard.
- [ ] Agent state persisted to external store (Redis/DynamoDB) — not in-process memory.
- [ ] Repeated tool call detection: same tool + same args N times → terminate with error.
- [ ] Fallback LLM defined for primary model failure.

### Error Handling
- [ ] `tenacity` retry decorators on all LLM and embedding calls.
- [ ] Circuit breaker present for external AI provider calls.
- [ ] AI-specific errors typed correctly (`EmbeddingFailureError`, `LLMTimeoutError`, etc.).
- [ ] Raw LLM error messages not surfaced to API responses.

---

## V. FLUTTER — Specific Review Rules

### Architecture
- [ ] No business logic in widgets — logic lives in use cases and state notifiers.
- [ ] No direct API calls from widgets — repository pattern enforced.
- [ ] `Either<Failure, T>` returned from repository methods — both sides handled in UI.
- [ ] Riverpod providers scoped appropriately — no global state for feature-specific data.

### UI / Performance
- [ ] `const` constructors used wherever possible.
- [ ] `ListView.builder` used for lists — never `ListView` with fixed children for dynamic data.
- [ ] No `setState` in production screens — state managed via Riverpod.
- [ ] `cachedNetworkImage` used for all remote images.
- [ ] Loading, success, and error states all handled in every async UI surface.

### Security
- [ ] Sensitive tokens stored in `flutter_secure_storage` — not `SharedPreferences`.
- [ ] No API keys or secrets in Dart code or asset files.
- [ ] Release build uses `--obfuscate --split-debug-info`.

---

## VI. AI SYSTEMS — Security-Specific Review Rules

### Prompt Injection
- [ ] User-controlled input is never directly interpolated into system prompts without sanitization.
- [ ] Retrieved RAG content is clearly delimited in prompts — cannot override system instructions.
- [ ] Agent tool descriptions do not include any instruction-like language that could be exploited.

### Data Privacy
- [ ] PII in RAG source documents is identified and classified before indexing.
- [ ] Retrieved chunks containing PII are handled with appropriate access control.
- [ ] Prompt + completion audit logs have PII scrubbed before storage.
- [ ] User-specific embeddings or personalized retrieval data are isolated per user — no cross-user data leakage.

### Model Governance
- [ ] LLM model version pinned in config — no dynamic model resolution.
- [ ] Model change requires an evaluation gate run before production promotion.
- [ ] Token usage per user session is bounded — no unbounded generation loops.

---

## VII. REVIEW SEVERITY DEFINITIONS

| Severity | Meaning | Action |
|---|---|---|
| P0 — Block | Security vulnerability, data corruption risk, legal/compliance issue | Must fix before merge. No exceptions. |
| P1 — Must Fix | Correctness bug, reliability gap, standards violation | Must fix before merge. |
| P2 — Should Fix | Performance issue, maintainability concern, missing test coverage | Fix in this PR or create tracked ticket. |
| P3 — Suggestion | Style, naming, optional improvement | Author's discretion. No blocking. |

---

Standards Version: v1.0
