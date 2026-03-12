# Tech Stack Playbook
# Spring Boot + Flutter + Python + RAG + Orchestration + AWS

Standards Version: v1.0
Applies to: All ShopConnect system designs unless stack is explicitly overridden.

---

## I. BACKEND SERVICES — Spring Boot (Java)

### Service Structure
- Follow Domain-Driven Design boundaries. One bounded context = one service.
- Use Spring Boot 3.x with Java 17+.
- All service entry points must expose `/actuator/health` and `/actuator/readiness`.
- Use `@Transactional` with explicit propagation rules. Never rely on default propagation in service layers.
- Prefer constructor injection. Avoid field injection (`@Autowired` on fields).

### API Layer
- All REST controllers must be versioned: `/api/v1/...`, `/api/v2/...`
- Request/Response DTOs must be separate from domain entities.
- Use `@Valid` on all request bodies. Validate at the controller boundary.
- Idempotency: all POST/PUT/PATCH endpoints must support `Idempotency-Key` header.
- OpenAPI 3.0 spec must be generated (Springdoc OpenAPI) and version-pinned.

### Security
- Spring Security with JWT-based stateless auth. No server-side session storage.
- Token validation on every request via filter chain.
- Role-based access: `@PreAuthorize` annotations on service methods.
- Secrets: inject via AWS Secrets Manager. Never hardcode. Never use application.properties for secrets.
- CORS configured explicitly. No wildcard origins in production.

### Data Access
- Use Spring Data JPA + Hibernate for relational data.
- Named queries or JPQL over raw SQL unless performance demands native queries.
- Every entity must define: primary key strategy, index annotations, and optimistic locking (`@Version`) for concurrent update patterns.
- Flyway for schema migrations. Every migration must have a corresponding rollback script.
- Use HikariCP connection pool. Configure `maximumPoolSize`, `connectionTimeout`, `idleTimeout` explicitly.

### Async / Messaging
- Use Spring Kafka or Spring SQS (AWS) for async event publishing.
- Never call external services synchronously in the main request path if latency SLO is tight.
- All Kafka/SQS consumers must be idempotent. Define dead-letter queue strategy.
- Outbox pattern for transactional event publishing (write to DB + publish atomically).

### Testing
- Unit: JUnit 5 + Mockito. Test service layer in isolation.
- Integration: `@SpringBootTest` + Testcontainers for DB and messaging.
- Contract: Spring Cloud Contract or Pact for API consumer-producer contracts.
- Coverage threshold: 80% minimum for service and repository layers.

---

## II. FRONTEND — Flutter

### Architecture
- Use Flutter 3.x (Dart 3+).
- State management: Riverpod (preferred) or BLoC. No `setState` in production screens.
- Clean architecture layers: presentation / domain / data. Strict layer isolation.
- Repository pattern for all data access. No direct API calls in widgets.

### API Integration
- Use `Dio` HTTP client with interceptors for auth token injection and retry logic.
- Define typed response models using `freezed` + `json_serializable`. No raw `Map<String, dynamic>` parsing.
- Handle network failures explicitly: timeout, 4xx, 5xx, and connectivity loss.
- All API calls must define a loading, success, and error state in the UI.

### Security
- Store tokens in `flutter_secure_storage`. Never in `SharedPreferences`.
- Certificate pinning for production API endpoints.
- Obfuscate release builds (`--obfuscate --split-debug-info`).

### Performance
- Lazy-load lists with pagination. Never load unbounded datasets.
- Use `const` constructors wherever possible to reduce rebuild scope.
- Image caching with `cached_network_image`. No raw `Image.network` in production.
- Define frame rate targets: 60fps on mid-tier devices as baseline NFR.

---

## III. AI SERVICES — Python

### Service Structure
- FastAPI (preferred) or Flask for Python AI microservices.
- Use async handlers (`async def`) for all I/O-bound AI service endpoints (embedding calls, LLM calls, vector DB queries).
- Each Python AI service must define a `/health` and `/ready` endpoint.
- Pydantic models for all request/response validation. No raw dict I/O.

### Dependencies & Environment
- Use `pyproject.toml` + `poetry` for dependency management. Pin all dependency versions.
- Separate `requirements.txt` for production vs development.
- Never load model weights or large artifacts at import time. Lazy-initialize at first request or startup hook.
- Environment config via `pydantic-settings`. No `os.getenv()` scattered across code.

### Concurrency
- Use `uvicorn` + `gunicorn` for multi-worker deployment.
- For CPU-bound tasks (embedding generation at scale), use `ProcessPoolExecutor` or offload to a dedicated worker queue (Celery + Redis or SQS).
- Never block the async event loop with synchronous CPU-heavy operations.

### Error Handling
- All LLM/embedding calls must have retry logic with exponential backoff.
- Define circuit breaker patterns around external AI provider calls (OpenAI, Bedrock, etc.).
- Log structured JSON. Include `trace_id`, `span_id`, `model_id`, `latency_ms` in every AI service log entry.

---

## IV. RAG SYSTEM — Retrieval Augmented Generation

### Architecture Pattern
- RAG pipeline must be decomposed into independent, replaceable stages:
  1. **Ingestion** — chunking, embedding, indexing
  2. **Retrieval** — query embedding, vector search, re-ranking
  3. **Generation** — prompt assembly, LLM call, response parsing
  4. **Evaluation** — offline + online metric collection

- Stages must be independently deployable and individually observable.
- Never couple ingestion to retrieval in the same service.

### Chunking Strategy
- Define explicit chunking strategy per document type. Not all content should be chunked identically.
- Minimum viable metadata per chunk: `document_id`, `chunk_index`, `source_uri`, `created_at`, `content_hash`.
- Overlapping chunks (10–15%) for context preservation at boundaries.
- Max chunk size must be tuned against embedding model context window. Default: 512 tokens with 50-token overlap.

### Embedding
- Specify embedding model explicitly. Never use a default without justification.
- Embedding model must be version-pinned. Re-embed all documents on model upgrade.
- Batch embedding during ingestion. Never embed one document at a time in production.
- Store embeddings with `vector_id` → `chunk_id` mapping for traceability.

### Vector Database
- Supported: pgvector (PostgreSQL), Pinecone, Weaviate, OpenSearch kNN.
- Define index type explicitly: HNSW (speed/recall tradeoff) vs IVFFlat (memory efficiency).
- Define `ef_construction`, `m` parameters for HNSW. Do not accept defaults in production.
- Index must support filtered search (metadata pre-filtering). Define filter fields at index creation.

### Retrieval
- Define top-K as a tunable parameter, not a hardcoded constant.
- Apply re-ranking (cross-encoder or reciprocal rank fusion) before passing context to LLM.
- Retrieval latency SLO must be defined: default target ≤ 150ms at p95.
- Hybrid retrieval (dense + sparse BM25) preferred for mixed keyword/semantic queries.

### Prompt Engineering
- System prompt and user prompt must be version-controlled separately.
- Context injection format must be consistent: use delimiters (`<context>`, `</context>`) for reliable parsing.
- Define max context token budget per prompt. Account for system prompt + retrieved chunks + response buffer.
- Never concatenate raw chunks without a structured separator.

### RAG Evaluation
- Offline metrics must be defined: Faithfulness, Answer Relevance, Context Recall, Context Precision.
- Use RAGAS or equivalent framework for automated RAG eval.
- Online metrics: user feedback signals (thumbs up/down), correction rate, session abandonment after RAG response.
- Establish evaluation dataset (golden set) for regression testing before any model or chunking change.

### Hallucination & Guardrails
- LLM output must be grounded to retrieved context. Define grounding check strategy.
- Implement output validation: structured response parsing with fallback for malformed output.
- For high-stakes domains (medical, financial, legal within ShopConnect): require citation of retrieved source in response.

---

## V. AI ORCHESTRATION

### Orchestration Architecture Patterns
- Choose one: **Pipeline** (deterministic, sequential), **DAG** (parallel branches), **Agent Loop** (LLM-driven tool selection), or **Hybrid**.
- For ShopConnect: prefer Pipeline for structured meal planning flows; Agent Loop for open-ended user queries.
- Never use an Agent Loop for workflows where output determinism is required.

### Orchestration Frameworks
- LangChain: use for chain composition and tool integration. Pin version. Avoid deprecated `LLMChain` patterns.
- LangGraph: preferred for stateful agent workflows with conditional branching and human-in-the-loop.
- Custom orchestrators: acceptable if framework lock-in is a concern. Must implement equivalent tracing and state management.
- Haystack: consider for RAG-heavy pipelines with structured component graphs.

### Tool / Function Calling
- All tools exposed to an LLM agent must have: name, description, typed input schema (JSON Schema), and output schema.
- Validate tool inputs before execution. Never pass raw LLM output directly to a function.
- Define tool execution timeout. Agents must not block indefinitely on tool calls.
- Log every tool call: `tool_name`, `input_args`, `output_summary`, `latency_ms`, `success`.

### Agent State Management
- Persist agent state (conversation history, tool results, intermediate outputs) to durable storage, not in-memory only.
- For multi-turn agents: use a session store (Redis or DynamoDB) keyed by `session_id`.
- Define max turn count per session. Agents must terminate or escalate, not loop indefinitely.
- Implement retry budget per session: max 3 tool call retries before graceful degradation.

### Prompt Chain Governance
- Every prompt in a chain must be version-controlled with a `prompt_id` and `prompt_version`.
- Prompt changes must go through an evaluation gate (offline eval before promotion to production).
- A/B test prompt versions before full rollout. Define success metric and rollback trigger.

### Observability for AI Workflows
- Trace every orchestration run end-to-end: `run_id`, `step_name`, `model_id`, `token_usage`, `latency_ms`.
- Use LangSmith, Langfuse, or equivalent for AI-specific tracing. Must not rely solely on generic APM.
- Track token consumption per run and per user session. Alert on anomalous token spend.
- Capture full prompt + completion for audit log (with PII scrubbing before storage).

### Failure Handling in Orchestration
- Define fallback chain: if primary LLM unavailable, route to secondary model or return degraded response.
- Implement timeout at the orchestration level (not just individual step level).
- Detect and break LLM hallucination loops: if agent selects the same tool with the same args 3x, terminate with error.
- All orchestration failures must emit a structured error event to the monitoring system.

---

## VI. AWS INFRASTRUCTURE

### Compute
- Spring Boot services: ECS Fargate (preferred for containerized workloads). EKS for complex multi-service orchestration needs.
- Python AI services: ECS Fargate for long-running services; Lambda for stateless, event-triggered functions (max 15min).
- Define task CPU/memory explicitly per service. Use right-sizing — do not default to max.

### Networking
- VPC with public/private subnet split. AI services and databases in private subnets only.
- ALB for HTTP/HTTPS traffic. NLB for low-latency TCP/UDP.
- Security Groups: least privilege. Explicitly define ingress/egress per service.
- VPC Endpoints for S3, SQS, Secrets Manager to avoid internet routing.

### Data Storage
- RDS PostgreSQL (with pgvector extension for embeddings if using in-DB vector search).
- Aurora Serverless v2 for variable-load workloads.
- S3 for document storage (RAG source documents, model artifacts, exports).
- ElastiCache Redis for session state, agent memory, rate limiting, short-TTL cache.
- DynamoDB for high-throughput key-value access (user preferences, session state at scale).

### AI/ML Services
- Amazon Bedrock for managed LLM access (Claude, Titan, Llama). Prefer Bedrock over direct OpenAI API for AWS-native deployments.
- SageMaker for custom model hosting if fine-tuned models are required.
- OpenSearch Service for hybrid vector + full-text search if pgvector is insufficient at scale.

### Messaging
- SQS for decoupled async processing (standard queue for at-least-once, FIFO for ordered workflows).
- SNS + SQS fan-out for event broadcasting across services.
- MSK (Managed Kafka) for high-throughput event streaming (e.g., user behavior events, real-time inventory updates).
- Define DLQ for every consumer queue.

### Secrets & Config
- AWS Secrets Manager for all credentials, API keys, DB passwords.
- Parameter Store for non-secret config values (feature flags, tuning parameters).
- Never pass secrets as environment variables in plain text in task definitions.

### Observability Stack
- CloudWatch Logs: structured JSON logs from all services.
- CloudWatch Metrics: custom metrics for AI-specific KPIs (retrieval latency, LLM token usage, RAG hit rate).
- X-Ray: distributed tracing across Spring Boot, Python, and Lambda services.
- CloudWatch Alarms: alert thresholds defined for p95 latency, error rate, queue depth, and token spend anomalies.

### Cost Controls
- Enable Cost Explorer + Budgets alerts per environment (dev/stage/prod).
- ECS Fargate Spot for non-critical and batch workloads.
- S3 Intelligent-Tiering for document storage. Define lifecycle policies.
- Reserved Instances or Savings Plans for predictable baseline workloads.
- Tag all resources with: `env`, `service`, `team`, `cost-center`.

---

## STACK OVERRIDE POLICY

If the problem statement specifies a different stack (e.g., Node.js backend, React frontend, GCP), override only the specified components and retain the non-overridden defaults from this playbook.

Explicit overrides must be stated in the Assumptions section of the design document.

---

Standards Version: v1.0
