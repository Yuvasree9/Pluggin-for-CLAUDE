# Implementation Patterns Playbook
# Spring Boot + Python + Flutter + RAG + Orchestration

Standards Version: v1.0
Applies to: All ShopConnect implementation work unless stack is explicitly overridden.

---

## I. SPRING BOOT SERVICE — Project Structure

```
service-name/
  src/
    main/
      java/com/shopconnect/<domain>/
        api/                     # Controllers (REST entry points only)
          v1/
        application/             # Use cases / application services
        domain/                  # Core business logic, entities, value objects
          model/
          repository/            # Repository interfaces (ports)
          service/               # Domain services
          event/                 # Domain events
        infrastructure/          # Adapters (DB, messaging, external APIs)
          persistence/           # JPA entities, Spring Data repos
          messaging/             # Kafka/SQS producers + consumers
          client/                # External API clients (Feign, RestTemplate)
          config/                # Spring config classes
        shared/                  # DTOs, mappers, exceptions, utils
      resources/
        db/migration/            # Flyway scripts (V1__init.sql, V2__...)
        application.yml
        application-dev.yml
        application-prod.yml
    test/
      java/com/shopconnect/<domain>/
        unit/
        integration/
        contract/
```

### Naming Conventions
- Controller: `<Entity>Controller` — handles HTTP only, delegates to application service
- Application service: `<Entity>Service` — orchestrates use case, no business logic
- Domain service: `<Entity>DomainService` — encapsulates complex domain rules
- Repository: `<Entity>Repository` (interface) + `<Entity>RepositoryImpl` (infra)
- DTO: `<Entity>Request`, `<Entity>Response`, `<Entity>Dto`
- Mapper: `<Entity>Mapper` (MapStruct preferred)
- Exception: `<Entity>NotFoundException`, `<Entity>ConflictException`

### Key Patterns
- **Repository Pattern**: Domain layer defines interface; infra layer implements with JPA.
- **Outbox Pattern**: Write domain event to `outbox` table in same DB transaction; relay publishes to Kafka/SQS asynchronously.
- **Idempotency**: Store `idempotency_key` + `response_hash` in a dedicated table. Return cached response on duplicate key.
- **Optimistic Locking**: Add `@Version Long version` to all entities updated concurrently.
- **Pagination**: Always return `Page<T>` from list endpoints. Never return unbounded lists.

### Error Handling
- Global `@ControllerAdvice` for all exception-to-HTTP status mapping.
- Domain exceptions extend `DomainException`. Application maps to HTTP status.
- Error response format:
  ```json
  {
    "error_code": "ITEM_NOT_FOUND",
    "message": "Item with id 123 not found",
    "trace_id": "abc-123",
    "timestamp": "2024-01-01T00:00:00Z"
  }
  ```
- Never expose stack traces in API responses.

### Logging
- Structured JSON via Logback + logstash-logback-encoder.
- Mandatory fields per log entry: `trace_id`, `span_id`, `service_name`, `env`, `level`, `message`.
- MDC: populate `trace_id` and `user_id` at request filter level.
- Never log PII (email, name, phone, payment data) without masking.

---

## II. PYTHON AI SERVICE — Project Structure

```
service-name/
  app/
    api/
      v1/
        routes/              # FastAPI routers (thin layer only)
        schemas/             # Pydantic request/response models
    core/
      config.py              # pydantic-settings config
      logging.py             # Structured JSON logger setup
      dependencies.py        # FastAPI dependency injection
    domain/
      models/                # Domain models (Pydantic BaseModel or dataclass)
      services/              # Business logic
      repositories/          # Repository interfaces
    infrastructure/
      db/                    # SQLAlchemy models + session
      vector_store/          # Vector DB client wrappers
      llm/                   # LLM client wrappers (Bedrock, OpenAI)
      cache/                 # Redis client
      messaging/             # SQS/Kafka producers + consumers
    rag/
      ingestion/             # Chunker, embedder, indexer
      retrieval/             # Query embedder, searcher, re-ranker
      generation/            # Prompt assembler, LLM caller, parser
      evaluation/            # RAGAS eval, golden set runner
    orchestration/
      agents/                # Agent definitions
      tools/                 # Tool implementations
      workflows/             # Pipeline / DAG / Agent loop definitions
      memory/                # Agent state / session management
  tests/
    unit/
    integration/
    eval/                    # RAG + agent evaluation tests
  Dockerfile
  pyproject.toml
```

### Naming Conventions
- Router: `<entity>_router.py`
- Schema: `<Entity>Request`, `<Entity>Response`
- Service: `<Entity>Service`
- Repository: `<Entity>Repository` (abstract) + `<Entity>RepositoryImpl`
- Tool: `<tool_name>_tool.py` — one tool per file

### Key Patterns
- **Dependency Injection**: Use FastAPI `Depends()` for services, DB sessions, LLM clients.
- **Async all the way**: All routes and service calls involving I/O must be `async def`.
- **Repository Pattern**: Identical to Spring Boot — domain owns interface, infra owns impl.
- **Retry with Backoff**: Use `tenacity` for LLM/embedding calls. Config: 3 retries, exponential backoff (1s, 2s, 4s), jitter enabled.
- **Circuit Breaker**: Use `pybreaker` or custom implementation around external AI provider calls.

### RAG — Ingestion Implementation
```python
# Standard ingestion flow
class DocumentIngestionService:
    async def ingest(self, document: Document) -> None:
        chunks = self.chunker.chunk(document, size=512, overlap=50)
        embeddings = await self.embedder.batch_embed(chunks)  # always batch
        await self.vector_store.upsert(embeddings, metadata=chunk_metadata)
        await self.outbox.publish(DocumentIndexedEvent(document_id=document.id))
```

### RAG — Retrieval Implementation
```python
# Standard retrieval flow
class RetrievalService:
    async def retrieve(self, query: str, top_k: int, filters: dict) -> list[Chunk]:
        query_embedding = await self.embedder.embed(query)
        candidates = await self.vector_store.search(query_embedding, top_k=top_k*3, filters=filters)
        reranked = await self.reranker.rerank(query, candidates, top_k=top_k)
        return reranked
```

### RAG — Generation Implementation
```python
# Standard generation flow
class GenerationService:
    async def generate(self, query: str, chunks: list[Chunk]) -> GenerationResult:
        context = self.prompt_assembler.assemble(chunks)  # structured, not raw concat
        prompt = self.prompt_registry.get(prompt_id="v2.1", query=query, context=context)
        response = await self.llm_client.complete(prompt, model=self.config.llm_model)
        validated = self.output_validator.validate(response)  # never skip
        return GenerationResult(answer=validated.text, sources=chunks, trace_id=...)
```

### Orchestration — Tool Implementation
```python
# Every tool must follow this contract
class PantryDepletionTool(BaseTool):
    name = "check_pantry_depletion"
    description = "Checks depletion probability for a given item in a user's pantry"
    input_schema = PantryDepletionInput  # Pydantic model
    output_schema = PantryDepletionOutput
    timeout_seconds = 5

    async def execute(self, input: PantryDepletionInput) -> PantryDepletionOutput:
        validated = PantryDepletionInput.model_validate(input)  # always validate
        result = await self.pantry_service.get_depletion(validated.user_id, validated.item_id)
        return PantryDepletionOutput(probability=result.probability, days_remaining=result.days)
```

### Error Handling
- All exceptions caught at router level via FastAPI exception handlers.
- Structured error response identical to Spring Boot format for consistency.
- AI-specific errors: `EmbeddingFailureError`, `LLMTimeoutError`, `RetrievalEmptyError`, `PromptValidationError`.
- Never surface raw LLM errors to the API caller.

### Logging
- `structlog` with JSON renderer.
- Mandatory fields: `trace_id`, `span_id`, `service`, `env`, `model_id`, `latency_ms` (for AI calls).
- Log every LLM call: prompt token count, completion token count, latency, model version.

---

## III. FLUTTER CLIENT — Project Structure

```
lib/
  core/
    config/          # App config, env vars
    di/              # Dependency injection (Riverpod providers)
    error/           # Failure types, error handling
    network/         # Dio setup, interceptors, retry logic
    router/          # GoRouter configuration
    theme/           # App theme, colors, typography
  features/
    <feature>/
      data/
        datasources/  # Remote + local data sources
        models/       # JSON-serializable API models (freezed)
        repositories/ # Repository implementations
      domain/
        entities/     # Pure Dart domain models
        repositories/ # Repository interfaces (abstract)
        usecases/     # Single-responsibility use cases
      presentation/
        pages/        # Full screens
        widgets/      # Feature-specific widgets
        providers/    # Riverpod state providers
        state/        # StateNotifier / AsyncNotifier classes
  shared/
    widgets/          # App-wide reusable widgets
    utils/
  main.dart
```

### Naming Conventions
- Page: `<Feature>Page`
- Widget: `<Purpose>Widget`
- Provider: `<feature><Entity>Provider`
- Use case: `Get<Entity>UseCase`, `Create<Entity>UseCase`
- Model: `<Entity>Model` (data layer), `<Entity>` (domain layer)
- Repository: `<Entity>Repository` (abstract), `<Entity>RepositoryImpl`

### State Management (Riverpod)
```dart
// AsyncNotifier pattern for all async operations
@riverpod
class MealPlanNotifier extends _$MealPlanNotifier {
  @override
  Future<MealPlan> build(String userId) async {
    return ref.watch(mealPlanRepositoryProvider).getMealPlan(userId);
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() =>
        ref.read(mealPlanRepositoryProvider).getMealPlan(arg));
  }
}
```

### Error Handling
- All repository methods return `Either<Failure, T>` (using `fpdart` or `dartz`).
- Failure types: `NetworkFailure`, `ServerFailure`, `CacheFailure`, `ValidationFailure`.
- UI always handles loading, success, and error states explicitly — no silent failures.

---

## IV. DATABASE — SQL Conventions

### Schema Rules
- All tables: `snake_case` naming.
- Primary key: `id UUID DEFAULT gen_random_uuid()` — no auto-increment integers for distributed systems.
- All tables: `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
- Soft deletes: `deleted_at TIMESTAMPTZ NULL` — never hard delete user data.
- Optimistic locking: `version BIGINT NOT NULL DEFAULT 0`.

### Index Strategy
- Every foreign key column must have an index.
- Columns used in WHERE clauses of frequent queries: add index.
- JSONB columns queried by key: use GIN index.
- Vector columns: use HNSW or IVFFlat index (pgvector).
- Partial indexes for filtered queries (e.g., `WHERE deleted_at IS NULL`).

### Migration Rules (Flyway)
- Naming: `V{version}__{description}.sql` (e.g., `V3__add_pantry_depletion_index.sql`)
- Every migration must have a paired `U{version}__undo_{description}.sql` rollback script.
- Additive-only changes preferred (add columns, add indexes) — never drop in same migration as create.
- Column rename: add new column → backfill → update app → drop old column across 3 separate deployments.

---

## V. INFRASTRUCTURE AS CODE — Conventions

### AWS CDK / Terraform Rules
- One stack per environment per service. No shared stacks across environments.
- All resource names include: `{service}-{env}-{resource-type}` (e.g., `pantry-ai-prod-ecs-task`).
- Secrets: reference from Secrets Manager by ARN — never hardcode values.
- IAM: one role per service, least-privilege policy. No wildcard `*` actions in production.
- ECS Task Definition: explicit CPU/memory, health check command, log configuration, and secrets injection from Secrets Manager.
- All infrastructure changes: plan → review → apply. No `terraform apply -auto-approve` in production.

---

Standards Version: v1.0
