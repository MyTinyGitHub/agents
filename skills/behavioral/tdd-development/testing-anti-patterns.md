---
name: test-driven-development
description: >
  Use when implementing any feature or bugfix in Java. Defines the TDD cycle
  for a JUnit 5 + Mockito + Spring Boot stack. Write the failing test first —
  always. No production code without a failing test preceding it.
---

Write the test first. Watch it fail. Write minimal code to pass.

**Core principle:** If you didn't watch the test fail, you don't know if it tests the right thing.

**Violating the letter of the rules is violating the spirit of the rules.**

## When to Use

**Always:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (ask your human partner):**
- Throwaway prototypes
- Generated boilerplate (DTOs, mappers)
- Configuration files

Thinking "skip TDD just this once"? Stop. That's rationalization.

## The Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

Wrote code before the test? Delete it. Start over.

**No exceptions:**
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete

Implement fresh from tests. Period.

## Red-Green-Refactor

```
RED   → Write one failing test
VERIFY RED   → Watch it fail for the right reason
GREEN  → Write minimal code to pass
VERIFY GREEN → All tests pass
REFACTOR → Clean up, stay green
REPEAT
```

## Stack

| Concern | Tool |
|---|---|
| Test framework | JUnit 5 (`@Test`, `@BeforeEach`, `@AfterEach`, `@Nested`) |
| Mocking | Mockito (`@Mock`, `@InjectMocks`, `@ExtendWith(MockitoExtension.class)`) |
| Assertions | JUnit 5 `Assertions` + Mockito `verify()` |
| Integration tests | `@SpringBootTest` |
| Run single test (Maven) | `mvn test -Dtest=ClassName#methodName` |
| Run all tests (Maven) | `mvn test` |

## RED — Write Failing Test

Write one minimal test showing what should happen.
Structure every test as **given / when / then**.

<Good>
```java
@ExtendWith(MockitoExtension.class)
class RetryServiceTest {

    @Mock
    private ExternalClient externalClient;

    @InjectMocks
    private RetryService retryService;

    @Test
    void retriesFailedOperationThreeTimes() {
        // given
        when(externalClient.call())
            .thenThrow(new RuntimeException("fail"))
            .thenThrow(new RuntimeException("fail"))
            .thenReturn("success");

        // when
        String result = retryService.callWithRetry();

        // then
        assertThat(result).isEqualTo("success");
        verify(externalClient, times(3)).call();
    }
}
```
Clear name, tests real behavior, one thing, given/when/then
</Good>

<Bad>
```java
@Test
void test1() {
    // when
    retryService.callWithRetry();

    // then
    verify(externalClient).call();
}
```
Vague name, tests mock interaction not behavior, missing given
</Bad>

**Requirements:**
- One behavior per test
- Method name describes the behavior (`retriesFailedOperationThreeTimes`, not `testRetry`)
- given / when / then sections always present as comments
- Real code — mock only external dependencies, not the class under test

## Verify RED — Watch It Fail

**MANDATORY. Never skip.**

```bash
mvn test -Dtest=RetryServiceTest#retriesFailedOperationThreeTimes
```

Or run the single test method from the IDE.

Confirm:
- Test fails (not compile error, not setup error — a real assertion failure)
- Failure message matches what you expected
- Fails because the feature is missing, not because of a typo

**Test passes immediately?** You are testing existing behavior or the test is wrong. Fix the test.

**Test errors on setup?** Fix the error, re-run until it fails correctly.

## GREEN — Minimal Code

Write the simplest code that makes the test pass.

<Good>
```java
public class RetryService {

    private final ExternalClient externalClient;

    public RetryService(ExternalClient externalClient) {
        this.externalClient = externalClient;
    }

    public String callWithRetry() {
        for (int i = 0; i < 3; i++) {
            try {
                return externalClient.call();
            } catch (RuntimeException e) {
                if (i == 2) throw e;
            }
        }
        throw new IllegalStateException("unreachable");
    }
}
```
Just enough to pass the test
</Good>

<Bad>
```java
public String callWithRetry(int maxRetries, BackoffStrategy backoff,
                             RetryListener listener, Duration timeout) {
    // YAGNI — no test requires this
}
```
Over-engineered beyond what the test demands
</Bad>

Do not add features, configuration options, or "improvements" the test does not require.

## Verify GREEN — Watch It Pass

**MANDATORY.**

```bash
mvn test -Dtest=RetryServiceTest
```

Confirm:
- The new test passes
- All other tests still pass
- No warnings or errors in output

**New test fails?** Fix the production code, not the test.

**Other tests break?** Fix them now before continuing.

## REFACTOR — Clean Up

After green only:
- Remove duplication
- Improve names
- Extract private helpers
- Apply project conventions (constructor injection, layer boundaries)

Keep all tests green throughout. Do not add new behavior during refactor.

## Test Naming Convention

Method names describe behavior, not implementation:

| Good | Bad |
|---|---|
| `throwsExceptionWhenEmailIsEmpty` | `testEmail` |
| `returnsEmptyListWhenNoResultsFound` | `testGetResults` |
| `updatesStatusToActiveWhenApproved` | `test1` |

If the name contains "and" — split into two tests.

## What to Mock

| Mock | Don't Mock |
|---|---|
| External HTTP clients | The class under test |
| Repository interfaces | Domain/service logic |
| Kafka producers/consumers | Simple value objects |
| Clock / time sources | Internal collaborators you own |

See `@anti-patterns.md` for mock anti-patterns.

## Integration Tests (`@SpringBootTest`)

Use `@SpringBootTest` only when:
- Testing the full Spring context wiring
- Testing database interactions end-to-end
- Testing HTTP layer behavior (`@WebMvcTest` preferred for controllers)

Do not use `@SpringBootTest` as a substitute for writing proper unit tests with Mockito.
Integration tests are slow — unit tests are the default.

## Verification Checklist

Before marking any work complete:

- [ ] Every new method has a corresponding test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for the expected reason (missing feature, not compile error)
- [ ] Wrote minimal code to pass — no YAGNI
- [ ] All tests pass (`mvn test`)
- [ ] Tests follow given / when / then structure
- [ ] Test names describe behavior
- [ ] Mocks used only for external dependencies

Can't check all boxes? You skipped TDD. Start over.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Too simple to test" | Simple code breaks. Test takes 2 minutes. |
| "I'll write tests after" | Tests written after pass immediately. That proves nothing. |
| "I already tested it manually" | Manual testing is ad-hoc. Can't re-run on every change. |
| "Deleting X hours of work is wasteful" | Sunk cost fallacy. Untested code is technical debt. |
| "TDD is dogmatic" | TDD is faster than debugging in production. |
| "Keep as reference, write tests around it" | You'll adapt it. That's testing after. Delete means delete. |

## Red Flags — STOP and Start Over

- Production code written before any test
- Test added after implementation is complete
- Test passes immediately without any implementation
- Can't explain why the test failed
- "I'll add tests later"
- "This case is too simple to test"
- "I already manually verified it"
- "Just this once"

**All of these mean: Delete the code. Start over with a failing test.**
