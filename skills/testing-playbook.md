# Testing Playbook
# Spring Boot + Python + Flutter + RAG + Orchestration

Standards Version: v1.0

---

## I. TESTING PHILOSOPHY

- Tests are first-class production artifacts. Untested code is unshippable code.
- Test at the right level: unit tests for logic, integration tests for boundaries, contract tests for APIs, E2E for critical flows.
- AI systems require a fourth dimension: evaluation tests (RAG quality, agent behavior, model drift).
- Coverage thresholds are minimums, not targets. High coverage with poor assertions is worthless.

---

## II. SPRING BOOT — Testing Standards

### Unit Tests (JUnit 5 + Mockito)
- Test service and domain layer in isolation. No Spring context in unit tests.
- Naming: `<ClassUnderTest>_<method>_<scenario>_<expectedBehavior>` (e.g., `PantryService_checkDepletion_whenItemMissing_throwsNotFoundException`)
- Mock all dependencies with `@ExtendWith(MockitoExtension.class)`.
- Use `assertThrows` for exception cases. Never catch and assert manually.
- Coverage threshold: 80% line coverage on service + domain packages.

```java
@ExtendWith(MockitoExtension.class)
class PantryServiceTest {
    @Mock PantryRepository pantryRepository;
    @InjectMocks PantryService pantryService;

    @Test
    void checkDepletion_whenItemExists_returnsDepletionProbability() {
        var item = PantryItemFixture.standard();
        when(pantryRepository.findById(item.getId())).thenReturn(Optional.of(item));

        var result = pantryService.checkDepletion(item.getId());

        assertThat(result.getProbability()).isGreaterThanOrEqualTo(0.0);
        verify(pantryRepository, times(1)).findById(item.getId());
    }
}
```

### Integration Tests (Testcontainers)
- Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full-stack integration.
- Always use Testcontainers for PostgreSQL, Redis, Kafka — never H2 or in-memory.
- Shared container lifecycle: use `@Container` + `@DynamicPropertySource` pattern.
- Each integration test must clean its own data. Use `@Transactional` or explicit teardown.
- Seed data via dedicated fixture builders, not raw SQL in test files.

```java
@SpringBootTest
@Testcontainers
class PantryControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }

    @Test
    void GET_pantry_item_returns200_withDepletionData() { ... }
}
```

### Contract Tests (Pact / Spring Cloud Contract)
- Every API consumed by another service must have a consumer-driven contract test.
- Consumer writes the contract (Pact). Provider verifies against it in CI.
- Contract failures block the provider's pipeline — no exceptions.
- Maintain contracts in a shared Pact Broker or contract repository.

### Performance Tests (Gatling / k6)
- Define baseline: p95 latency target and throughput from NFR document.
- Run performance tests in the CI stage gate for staging deployments only (not on every PR).
- Scenarios: steady load, ramp-up, spike, soak (overnight).
- Alert threshold: if p95 degrades > 20% vs baseline → block deployment.

---

## III. PYTHON AI SERVICES — Testing Standards

### Unit Tests (pytest)
- Naming: `test_<method>_<scenario>_<expected>` in `tests/unit/` directory.
- Use `pytest-asyncio` for async service tests.
- Mock external dependencies: LLM clients, vector DB, Redis, SQS — use `pytest-mock` or `unittest.mock.AsyncMock`.
- Never make real API calls in unit tests.
- Coverage threshold: 80% for service + domain packages.

```python
@pytest.mark.asyncio
async def test_retrieve_returns_reranked_chunks_when_query_matches(
    retrieval_service: RetrievalService,
    mock_vector_store: AsyncMock,
    mock_reranker: AsyncMock
):
    mock_vector_store.search.return_value = ChunkFixture.list(n=10)
    mock_reranker.rerank.return_value = ChunkFixture.list(n=3)

    result = await retrieval_service.retrieve(query="what milk do I need?", top_k=3)

    assert len(result) == 3
    mock_reranker.rerank.assert_awaited_once()
```

### Integration Tests (pytest + Testcontainers)
- Use `testcontainers-python` for PostgreSQL, Redis, pgvector.
- Test full RAG pipeline end-to-end with a small in-memory vector store or real pgvector container.
- Seed test data from fixtures. Clean up after each test session.

### RAG Evaluation Tests (RAGAS / Custom)
- Maintain a **golden evaluation set**: 50–100 curated query + expected answer pairs per domain.
- Run RAGAS metrics: Faithfulness, Answer Relevance, Context Recall, Context Precision.
- Minimum thresholds (adjust per domain):
  - Faithfulness ≥ 0.85
  - Answer Relevance ≥ 0.80
  - Context Recall ≥ 0.75
  - Context Precision ≥ 0.70
- Eval tests run in CI on every change to: chunking strategy, embedding model, prompt template, retrieval config.
- A regression (metric drops > 5%) blocks promotion to production.

```python
# Example RAGAS eval test structure
@pytest.mark.eval
async def test_rag_quality_meets_thresholds(rag_pipeline, golden_set):
    results = []
    for sample in golden_set:
        answer = await rag_pipeline.query(sample.question)
        results.append({"question": sample.question, "answer": answer.text,
                        "contexts": answer.contexts, "ground_truth": sample.expected})

    scores = evaluate(Dataset.from_list(results),
                      metrics=[faithfulness, answer_relevancy, context_recall, context_precision])

    assert scores["faithfulness"] >= 0.85, f"Faithfulness {scores['faithfulness']} below threshold"
    assert scores["answer_relevancy"] >= 0.80
    assert scores["context_recall"] >= 0.75
    assert scores["context_precision"] >= 0.70
```

### Agent / Orchestration Tests
- Test agent behavior with mock tools. Never use real tools in unit/integration tests.
- Scenarios to cover:
  - Happy path: agent completes goal in expected number of turns.
  - Tool failure: agent handles tool error gracefully and degrades.
  - Turn limit reached: agent terminates cleanly without infinite loop.
  - Ambiguous input: agent requests clarification rather than hallucinating.
- Log all agent test runs with full trace (turns, tool calls, token counts) for debugging.

---

## IV. FLUTTER — Testing Standards

### Widget Tests
- Test all custom widgets in isolation using `testWidgets`.
- Mock providers with `ProviderContainer` overrides (Riverpod).
- Test loading, success, and error states for every data-driven widget.
- Never test Flutter framework widgets directly — test your own widget logic.

```dart
testWidgets('PantryItemCard shows depletion warning when probability > 0.85', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [pantryItemProvider.overrideWith((ref, id) async =>
          PantryItemFixture.highDepletion())],
      child: const MaterialApp(home: PantryItemCard(itemId: 'test-id')),
    ),
  );
  await tester.pumpAndSettle();
  expect(find.byKey(Key('depletion-warning')), findsOneWidget);
});
```

### Unit Tests (Dart test)
- Test all use cases, repositories, and domain logic.
- Mock datasources with Mockito for Dart.
- Test `Either<Failure, T>` return paths — both left (failure) and right (success).

### Integration Tests
- Use `flutter_test` integration test runner.
- Cover critical user journeys: pantry check → meal plan generation → cart → checkout.
- Run on real device or emulator in CI (Firebase Test Lab or AWS Device Farm).

### Golden Tests
- Use `matchesGoldenFile` for pixel-sensitive UI components (product cards, charts, badges).
- Update golden files intentionally in PRs. Unintended golden file changes fail the build.

---

## V. CROSS-CUTTING TEST STANDARDS

### Test Data Management
- Use **fixture builders** (not raw JSON files) for all test data.
- Fixtures must produce valid domain objects by default, with overrideable fields.
- Never share mutable test state between tests.
- In-database fixtures: use dedicated test schemas or prefixed table names, never production-adjacent data.

### Test Environment Strategy
- `unit`: no infrastructure, all mocked — runs in < 1 minute
- `integration`: Testcontainers — runs in < 5 minutes
- `contract`: Pact broker — runs on PR and on provider deploy
- `eval`: RAG golden set — runs on AI-related changes only (skippable in feature PRs)
- `e2e`: real staging environment — runs on staging deploy only

### CI Pipeline Integration
```
PR gate:       unit + integration + contract
Staging gate:  unit + integration + contract + eval (if AI changed) + performance baseline
Prod gate:     smoke tests only + canary validation metrics
```

### Coverage Reporting
- Use JaCoCo (Spring Boot), coverage.py (Python), flutter test --coverage.
- Report to Codecov or SonarQube.
- Coverage drop of > 5% on a PR triggers a required review comment.
- Coverage thresholds enforced in CI — build fails below threshold.

---

Standards Version: v1.0
