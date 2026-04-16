---
name: idiomatic-java
description: >
  General rules for writing idiomatic, modern Java code. Use on any Java task —
  new features, refactoring, code review. Covers style, modern language features,
  null safety, naming, and layer boundaries. Targets Java 17/21 features where
  available, with fallback guidance for Java 11 projects.
---

Write Java that is minimal, expressive, and modern. The biggest failure modes
i---
name: idiomatic-java
description: >
  General rules for writing idiomatic, modern Java code. Use on any Java task —
  new features, refactoring, code review. Covers style, modern language features,
  null safety, naming, and layer boundaries. Targets Java 17/21 features where
  available, with fallback guidance for Java 11 projects.
---

Write Java that is minimal, expressive, and modern. The biggest failure modes
in agent-generated Java are verbosity, over-engineering, null blindness, poor
naming, and layer boundary violations. This skill addresses all five.

## The Core Rules

```
1. Use the most expressive modern construct available
2. Never add complexity the current requirement does not justify
3. Null must be handled explicitly — never ignored
4. Names must describe intent, not implementation
5. Code must respect layer boundaries
```

---

## 1. Prefer Modern Java — Stop Writing Java 8 in 2024

### Use Records for pure data carriers

```java
// ❌ BAD: verbose POJO
public class PolicySummary {
    private final String policyNumber;
    private final BigDecimal premium;

    public PolicySummary(String policyNumber, BigDecimal premium) {
        this.policyNumber = policyNumber;
        this.premium = premium;
    }
    // + getters, equals, hashCode, toString...
}

// ✅ GOOD: record
public record PolicySummary(String policyNumber, BigDecimal premium) {}
```

Use records for: DTOs, value objects, query results, method return bundles.
Do not use records for: mutable entities, JPA `@Entity` classes.

---

### Use pattern matching for `instanceof`

```java
// ❌ BAD
if (event instanceof PolicyCreatedEvent) {
    PolicyCreatedEvent created = (PolicyCreatedEvent) event;
    process(created.getPolicyNumber());
}

// ✅ GOOD
if (event instanceof PolicyCreatedEvent created) {
    process(created.getPolicyNumber());
}
```

---

### Use text blocks for multiline strings

```java
// ❌ BAD
String query = "SELECT p.id, p.number\n" +
               "FROM policy p\n" +
               "WHERE p.status = 'ACTIVE'";

// ✅ GOOD
String query = """
        SELECT p.id, p.number
        FROM policy p
        WHERE p.status = 'ACTIVE'
        """;
```

---

### Use Stream API for collection transformations

```java
// ❌ BAD: imperative loop
List<String> activeNumbers = new ArrayList<>();
for (Policy policy : policies) {
    if (policy.isActive()) {
        activeNumbers.add(policy.getNumber());
    }
}

// ✅ GOOD: stream pipeline
List<String> activeNumbers = policies.stream()
    .filter(Policy::isActive)
    .map(Policy::getNumber)
    .toList();
```

Rules for streams:
- Prefer method references over lambdas when the lambda just delegates (`Policy::isActive` not `p -> p.isActive()`)
- Use `.toList()` (Java 16+) instead of `.collect(Collectors.toList())`
- Do not use streams for side effects — use a for-each loop instead
- Do not chain more than ~4-5 operations without extracting a named method

---

## 2. Never Over-Engineer

Write the simplest solution that satisfies the current requirement.
Do not add abstractions, configuration, or extension points that no test requires.

```java
// ❌ BAD: generic, configurable, over-abstracted
public class RetryExecutor<T> {
    private final int maxRetries;
    private final Duration backoff;
    private final RetryListener<T> listener;
    private final Predicate<Exception> retryCondition;
    // ...
}

// ✅ GOOD: solves what is needed now
public class RetryService {
    public String callWithRetry(Supplier<String> operation) {
        for (int i = 0; i < 3; i++) {
            try {
                return operation.get();
            } catch (RuntimeException e) {
                if (i == 2) throw e;
            }
        }
        throw new IllegalStateException("unreachable");
    }
}
```

**Hard rules:**
- No design patterns unless the problem explicitly requires them
- No factory/builder/strategy unless there are at least two concrete variations right now
- No `Abstract`, `Base`, `Generic`, `Manager`, `Helper`, `Util` in class names unless unavoidable
- No configuration parameters the current requirement doesn't vary

---

## 3. Null Safety — Handle It, Don't Ignore It

### Use `Optional` for values that may be absent

```java
// ❌ BAD: returning null
public Policy findByNumber(String number) {
    return policyRepository.findByNumber(number); // may return null
}

// ✅ GOOD: return Optional
public Optional<Policy> findByNumber(String number) {
    return policyRepository.findByNumber(number);
}
```

**Optional rules:**
- Use `Optional` as a return type only — never as a field or parameter
- Never call `.get()` without `.isPresent()` — use `.orElseThrow()`, `.orElse()`, `.map()`, `.ifPresent()`
- Never wrap collections in `Optional` — return an empty collection instead
- Do not use `Optional.ofNullable()` defensively everywhere — fix the source of nulls

```java
// ❌ BAD: Optional as a field
private Optional<String> email;

// ❌ BAD: Optional as a parameter
public void process(Optional<Policy> policy) { ... }

// ✅ GOOD: Optional as return type
public Optional<Policy> find(String id) { ... }
```

### Fail fast on null inputs

```java
// ✅ GOOD: validate at entry points
public PolicyService(PolicyRepository repository) {
    this.repository = Objects.requireNonNull(repository, "repository must not be null");
}
```

Use `Objects.requireNonNull()` in constructors and public method entry points.
Use `@NonNull` / `@Nullable` annotations to document intent in method signatures.

---

## 4. Naming — Describe Intent, Not Implementation

Names must answer "what" not "how".

| Bad | Good | Why |
|---|---|---|
| `temp`, `temp1`, `data` | `activePolicies`, `requestPayload` | Describes what it holds |
| `doProcess()` | `calculatePremium()` | Describes the action |
| `flag`, `result`, `val` | `isExpired`, `discountedPrice` | Reveals meaning |
| `PolicyManager` | `PolicyService`, `PolicyRepository` | Manager is meaningless |
| `AbstractBasePolicy` | Extract an interface instead | Inheritance hierarchy smell |
| `handleIt()` | `rejectExpiredPolicy()` | Specific beats generic |

**Rules:**
- Booleans: prefix with `is`, `has`, `can`, `should` (`isActive`, `hasExpired`)
- Methods: start with a verb that describes the action (`calculate`, `find`, `create`, `validate`, `reject`)
- Collections: always plural (`policies`, not `policyList`, not `policyData`)
- No abbreviations unless universally understood (`id`, `url`, `dto` — fine. `plcNum` — not fine)
- No single-letter variables except loop indices and lambda parameters in short, obvious pipelines

---

## 5. Layer Boundaries — Code Belongs Where It Belongs

Each layer has a single responsibility. Crossing boundaries is a design error.

```
Controller  →  receives HTTP, delegates to Service, returns response
Service     →  owns business logic, orchestrates, calls Repository
Repository  →  data access only, no business logic
Domain      →  entities, value objects, domain rules — no Spring, no HTTP
```

**Hard rules:**

```java
// ❌ BAD: business logic in controller
@PostMapping("/policies")
public ResponseEntity<Void> create(@RequestBody PolicyRequest request) {
    if (request.getPremium().compareTo(BigDecimal.ZERO) <= 0) {
        return ResponseEntity.badRequest().build(); // validation belongs in service/domain
    }
    // ...
}

// ✅ GOOD: controller delegates
@PostMapping("/policies")
public ResponseEntity<Void> create(@RequestBody PolicyRequest request) {
    policyService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).build();
}
```

```java
// ❌ BAD: repository logic in service
public List<Policy> getActivePolicies() {
    return policyRepository.findAll().stream() // loading all to filter in memory
        .filter(Policy::isActive)
        .toList();
}

// ✅ GOOD: query belongs in repository
public List<Policy> getActivePolicies() {
    return policyRepository.findAllByStatus(PolicyStatus.ACTIVE);
}
```

- Controllers never contain `if` chains on business conditions
- Services never construct HTTP responses or know about `HttpStatus`
- Repositories never contain business rules — only queries
- Domain objects never import Spring annotations (`@Service`, `@Component`, etc.)

---

## Anti-Patterns

- **Primitive obsession** — passing raw `String` for policy numbers, `long` for IDs everywhere. Wrap in value types or records.
- **Anemic domain model** — entities are just bags of fields with getters/setters and all logic lives in services. Put behavior where the data is.
- **Exception swallowing** — `catch (Exception e) { log.error(...); }` with no rethrow. Either handle it or rethrow.
- **Boolean parameters** — `processPolicy(policy, true, false)`. Extract into named methods or an enum.
- **Long parameter lists** — more than 3-4 parameters is a signal to introduce a parameter object or record.
- **Mutable statics** — no shared mutable state. Ever.

---

## Quick Checklist

Before presenting any Java code:

- [ ] Could a record replace this class?
- [ ] Is there a stream pipeline that replaces this loop?
- [ ] Are all nullable return types wrapped in `Optional`?
- [ ] Does every name describe intent, not implementation?
- [ ] Is business logic in the service, not the controller or repository?
- [ ] Is there any abstraction, pattern, or parameter that no current requirement justifies?n agent-generated Java are verbosity, over-engineering, null blindness, poor
naming, and layer boundary violations. This skill addresses all five.

## The Core Rules

```
1. Use the most expressive modern construct available
2. Never add complexity the current requirement does not justify
3. Null must be handled explicitly — never ignored
4. Names must describe intent, not implementation
5. Code must respect layer boundaries
```

---

## 1. Prefer Modern Java — Stop Writing Java 8 in 2024

### Use Records for pure data carriers

```java
// ❌ BAD: verbose POJO
public class PolicySummary {
    private final String policyNumber;
    private final BigDecimal premium;

    public PolicySummary(String policyNumber, BigDecimal premium) {
        this.policyNumber = policyNumber;
        this.premium = premium;
    }
    // + getters, equals, hashCode, toString...
}

// ✅ GOOD: record
public record PolicySummary(String policyNumber, BigDecimal premium) {}
```

Use records for: DTOs, value objects, query results, method return bundles.
Do not use records for: mutable entities, JPA `@Entity` classes.

---

### Use pattern matching for `instanceof`

```java
// ❌ BAD
if (event instanceof PolicyCreatedEvent) {
    PolicyCreatedEvent created = (PolicyCreatedEvent) event;
    process(created.getPolicyNumber());
}

// ✅ GOOD
if (event instanceof PolicyCreatedEvent created) {
    process(created.getPolicyNumber());
}
```

---

### Use text blocks for multiline strings

```java
// ❌ BAD
String query = "SELECT p.id, p.number\n" +
               "FROM policy p\n" +
               "WHERE p.status = 'ACTIVE'";

// ✅ GOOD
String query = """
        SELECT p.id, p.number
        FROM policy p
        WHERE p.status = 'ACTIVE'
        """;
```

---

### Use Stream API for collection transformations

```java
// ❌ BAD: imperative loop
List<String> activeNumbers = new ArrayList<>();
for (Policy policy : policies) {
    if (policy.isActive()) {
        activeNumbers.add(policy.getNumber());
    }
}

// ✅ GOOD: stream pipeline
List<String> activeNumbers = policies.stream()
    .filter(Policy::isActive)
    .map(Policy::getNumber)
    .toList();
```

Rules for streams:
- Prefer method references over lambdas when the lambda just delegates (`Policy::isActive` not `p -> p.isActive()`)
- Use `.toList()` (Java 16+) instead of `.collect(Collectors.toList())`
- Do not use streams for side effects — use a for-each loop instead
- Do not chain more than ~4-5 operations without extracting a named method

---

## 2. Never Over-Engineer

Write the simplest solution that satisfies the current requirement.
Do not add abstractions, configuration, or extension points that no test requires.

```java
// ❌ BAD: generic, configurable, over-abstracted
public class RetryExecutor<T> {
    private final int maxRetries;
    private final Duration backoff;
    private final RetryListener<T> listener;
    private final Predicate<Exception> retryCondition;
    // ...
}

// ✅ GOOD: solves what is needed now
public class RetryService {
    public String callWithRetry(Supplier<String> operation) {
        for (int i = 0; i < 3; i++) {
            try {
                return operation.get();
            } catch (RuntimeException e) {
                if (i == 2) throw e;
            }
        }
        throw new IllegalStateException("unreachable");
    }
}
```

**Hard rules:**
- No design patterns unless the problem explicitly requires them
- No factory/builder/strategy unless there are at least two concrete variations right now
- No `Abstract`, `Base`, `Generic`, `Manager`, `Helper`, `Util` in class names unless unavoidable
- No configuration parameters the current requirement doesn't vary

---

## 3. Null Safety — Handle It, Don't Ignore It

### Use `Optional` for values that may be absent

```java
// ❌ BAD: returning null
public Policy findByNumber(String number) {
    return policyRepository.findByNumber(number); // may return null
}

// ✅ GOOD: return Optional
public Optional<Policy> findByNumber(String number) {
    return policyRepository.findByNumber(number);
}
```

**Optional rules:**
- Use `Optional` as a return type only — never as a field or parameter
- Never call `.get()` without `.isPresent()` — use `.orElseThrow()`, `.orElse()`, `.map()`, `.ifPresent()`
- Never wrap collections in `Optional` — return an empty collection instead
- Do not use `Optional.ofNullable()` defensively everywhere — fix the source of nulls

```java
// ❌ BAD: Optional as a field
private Optional<String> email;

// ❌ BAD: Optional as a parameter
public void process(Optional<Policy> policy) { ... }

// ✅ GOOD: Optional as return type
public Optional<Policy> find(String id) { ... }
```

### Fail fast on null inputs

```java
// ✅ GOOD: validate at entry points
public PolicyService(PolicyRepository repository) {
    this.repository = Objects.requireNonNull(repository, "repository must not be null");
}
```

Use `Objects.requireNonNull()` in constructors and public method entry points.
Use `@NonNull` / `@Nullable` annotations to document intent in method signatures.

---

## 4. Naming — Describe Intent, Not Implementation

Names must answer "what" not "how".

| Bad | Good | Why |
|---|---|---|
| `temp`, `temp1`, `data` | `activePolicies`, `requestPayload` | Describes what it holds |
| `doProcess()` | `calculatePremium()` | Describes the action |
| `flag`, `result`, `val` | `isExpired`, `discountedPrice` | Reveals meaning |
| `PolicyManager` | `PolicyService`, `PolicyRepository` | Manager is meaningless |
| `AbstractBasePolicy` | Extract an interface instead | Inheritance hierarchy smell |
| `handleIt()` | `rejectExpiredPolicy()` | Specific beats generic |

**Rules:**
- Booleans: prefix with `is`, `has`, `can`, `should` (`isActive`, `hasExpired`)
- Methods: start with a verb that describes the action (`calculate`, `find`, `create`, `validate`, `reject`)
- Collections: always plural (`policies`, not `policyList`, not `policyData`)
- No abbreviations unless universally understood (`id`, `url`, `dto` — fine. `plcNum` — not fine)
- No single-letter variables except loop indices and lambda parameters in short, obvious pipelines

---

## 5. Layer Boundaries — Code Belongs Where It Belongs

Each layer has a single responsibility. Crossing boundaries is a design error.

```
Controller  →  receives HTTP, delegates to Service, returns response
Service     →  owns business logic, orchestrates, calls Repository
Repository  →  data access only, no business logic
Domain      →  entities, value objects, domain rules — no Spring, no HTTP
```

**Hard rules:**

```java
// ❌ BAD: business logic in controller
@PostMapping("/policies")
public ResponseEntity<Void> create(@RequestBody PolicyRequest request) {
    if (request.getPremium().compareTo(BigDecimal.ZERO) <= 0) {
        return ResponseEntity.badRequest().build(); // validation belongs in service/domain
    }
    // ...
}

// ✅ GOOD: controller delegates
@PostMapping("/policies")
public ResponseEntity<Void> create(@RequestBody PolicyRequest request) {
    policyService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).build();
}
```

```java
// ❌ BAD: repository logic in service
public List<Policy> getActivePolicies() {
    return policyRepository.findAll().stream() // loading all to filter in memory
        .filter(Policy::isActive)
        .toList();
}

// ✅ GOOD: query belongs in repository
public List<Policy> getActivePolicies() {
    return policyRepository.findAllByStatus(PolicyStatus.ACTIVE);
}
```

- Controllers never contain `if` chains on business conditions
- Services never construct HTTP responses or know about `HttpStatus`
- Repositories never contain business rules — only queries
- Domain objects never import Spring annotations (`@Service`, `@Component`, etc.)

---

## Anti-Patterns

- **Primitive obsession** — passing raw `String` for policy numbers, `long` for IDs everywhere. Wrap in value types or records.
- **Anemic domain model** — entities are just bags of fields with getters/setters and all logic lives in services. Put behavior where the data is.
- **Exception swallowing** — `catch (Exception e) { log.error(...); }` with no rethrow. Either handle it or rethrow.
- **Boolean parameters** — `processPolicy(policy, true, false)`. Extract into named methods or an enum.
- **Long parameter lists** — more than 3-4 parameters is a signal to introduce a parameter object or record.
- **Mutable statics** — no shared mutable state. Ever.

---

## Quick Checklist

Before presenting any Java code:

- [ ] Could a record replace this class?
- [ ] Is there a stream pipeline that replaces this loop?
- [ ] Are all nullable return types wrapped in `Optional`?
- [ ] Does every name describe intent, not implementation?
- [ ] Is business logic in the service, not the controller or repository?
- [ ] Is there any abstraction, pattern, or parameter that no current requirement justifies?
