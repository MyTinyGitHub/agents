---
name: java-error-handling
description: >
  Rules for defensive, complete error handling in Java. Use on any Java task
  involving input validation, service logic, external calls, or exception
  propagation. Enforces: guard clauses at entry points, no silent failures,
  no swallowed exceptions, custom exceptions for domain errors, and
  @ControllerAdvice for HTTP error mapping. Uses Spring Assert for preconditions.
---

Every code path has an outcome — including the ones where things go wrong.
Handle all of them explicitly. Never assume the happy path is the only path.

## The Iron Law

```
1. Validate inputs at every entry point — before any logic runs
2. Never swallow exceptions silently
3. Every catch block either handles, wraps, or rethrows — never just logs
4. Domain errors get domain exceptions — not generic RuntimeException
5. The HTTP layer never leaks internal exceptions
```

---

## 1. Guard Clauses at Every Entry Point

Validate all inputs before any business logic runs.
Use **Spring Assert** for precondition checks — it is already on the classpath.

```java
// ❌ BAD: no validation, proceeds blindly
public Policy activate(String policyNumber) {
    Policy policy = policyRepository.findByNumber(policyNumber).orElseThrow();
    policy.activate();
    return policyRepository.save(policy);
}

// ✅ GOOD: guard clauses first
public Policy activate(String policyNumber) {
    Assert.hasText(policyNumber, "policyNumber must not be blank");

    Policy policy = policyRepository.findByNumber(policyNumber)
        .orElseThrow(() -> new PolicyNotFoundException(policyNumber));

    Assert.isTrue(policy.canBeActivated(),
        () -> "Policy %s cannot be activated in state %s"
            .formatted(policyNumber, policy.getStatus()));

    policy.activate();
    return policyRepository.save(policy);
}
```

### Spring Assert — Core Methods

| Method | Throws | Use for |
|---|---|---|
| `Assert.notNull(obj, msg)` | `IllegalArgumentException` | Non-null parameters |
| `Assert.hasText(str, msg)` | `IllegalArgumentException` | Non-blank strings |
| `Assert.isTrue(expr, msg)` | `IllegalArgumentException` | Boolean preconditions |
| `Assert.state(expr, msg)` | `IllegalStateException` | Object state invariants |
| `Assert.notEmpty(col, msg)` | `IllegalArgumentException` | Non-empty collections |

Use **lambda message suppliers** for messages that involve string formatting —
the message is only built if the assertion fails:

```java
// ❌ BAD: message always built
Assert.isTrue(policy.isActive(), "Policy " + policyNumber + " is not active");

// ✅ GOOD: message built only on failure
Assert.isTrue(policy.isActive(),
    () -> "Policy %s is not active".formatted(policyNumber));
```

---

## 2. Custom Domain Exceptions

Do not throw raw `RuntimeException` or `IllegalArgumentException` for domain
errors — they carry no semantic meaning and are hard to handle specifically.

Define exceptions per domain concept:

```java
// Base application exception — all custom exceptions extend this
public class ApplicationException extends RuntimeException {
    public ApplicationException(String message) {
        super(message);
    }
    public ApplicationException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Domain-specific exceptions
public class PolicyNotFoundException extends ApplicationException {
    public PolicyNotFoundException(String policyNumber) {
        super("Policy not found: " + policyNumber);
    }
}

public class PolicyActivationException extends ApplicationException {
    public PolicyActivationException(String policyNumber, PolicyStatus currentStatus) {
        super("Policy %s cannot be activated in status %s"
            .formatted(policyNumber, currentStatus));
    }
}

public class DuplicatePolicyException extends ApplicationException {
    public DuplicatePolicyException(String policyNumber) {
        super("Policy already exists: " + policyNumber);
    }
}
```

**Rules:**
- One exception class per distinct domain error — not one generic `ServiceException`
- Exception message must include the relevant identifier (policy number, user id, etc.)
- Always provide a constructor that accepts a `cause` for wrapping lower-level exceptions
- Use `RuntimeException` (unchecked) — do not use checked exceptions for domain errors

---

## 3. Never Swallow Exceptions

Every catch block must do one of three things:
**handle it**, **wrap and rethrow it**, or **rethrow it**.
Logging alone is not handling.

```java
// ❌ BAD: swallowed — failure is invisible
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    log.error("Failed to save policy", e);
    // execution continues as if nothing happened
}

// ❌ BAD: logged and rethrown — exception logged twice (here and at the handler)
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    log.error("Failed to save policy", e);
    throw e;
}

// ✅ GOOD: wrap in domain exception and rethrow — logged once at the boundary
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    throw new PolicyPersistenceException(policy.getNumber(), e);
}

// ✅ GOOD: handle it completely — no rethrow needed
try {
    externalRatingService.rate(policy);
} catch (RatingServiceUnavailableException e) {
    log.warn("Rating service unavailable for policy {} — using default rate", policy.getNumber());
    policy.applyDefaultRate();
}
```

**Log once rule:** Log the exception at the layer that handles it — not at every
layer it passes through. If you rethrow, do not log.

---

## 4. Handle Optional Explicitly — Never .get() Blindly

```java
// ❌ BAD: NoSuchElementException with no context
Policy policy = policyRepository.findByNumber(number).get();

// ❌ BAD: NullPointerException waiting to happen
Policy policy = policyRepository.findByNumber(number).orElse(null);
doSomething(policy.getStatus()); // NPE

// ✅ GOOD: explicit domain exception
Policy policy = policyRepository.findByNumber(number)
    .orElseThrow(() -> new PolicyNotFoundException(number));

// ✅ GOOD: explicit fallback when absence is valid
Policy policy = policyRepository.findByNumber(number)
    .orElse(Policy.defaultPolicy());

// ✅ GOOD: presence is optional by design
policyRepository.findByNumber(number)
    .ifPresent(policy -> notificationService.notify(policy));
```

---

## 5. @ControllerAdvice — Map Exceptions to HTTP Responses

All exception-to-HTTP mapping lives in a single `@ControllerAdvice` class.
Controllers never contain try/catch blocks for domain exceptions.

```java
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(PolicyNotFoundException.class)
    public ResponseEntity<ErrorResponse> handlePolicyNotFound(PolicyNotFoundException e) {
        log.warn("Policy not found: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of(e.getMessage()));
    }

    @ExceptionHandler(DuplicatePolicyException.class)
    public ResponseEntity<ErrorResponse> handleDuplicatePolicy(DuplicatePolicyException e) {
        log.warn("Duplicate policy: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(ErrorResponse.of(e.getMessage()));
    }

    @ExceptionHandler(PolicyActivationException.class)
    public ResponseEntity<ErrorResponse> handleActivationError(PolicyActivationException e) {
        log.warn("Policy activation failed: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(ErrorResponse.of(e.getMessage()));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(IllegalArgumentException e) {
        log.warn("Invalid input: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse.of(e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of("An unexpected error occurred"));
    }
}
```

```java
// Standard error response body
@Value
@Builder
public class ErrorResponse {
    String message;

    public static ErrorResponse of(String message) {
        return ErrorResponse.builder().message(message).build();
    }
}
```

**Rules:**
- Every custom domain exception has a dedicated `@ExceptionHandler`
- The catch-all `Exception` handler always logs at `error` with the full throwable
- Domain exception handlers log at `warn` — they are expected, not surprising
- Never expose internal exception messages, stack traces, or class names to the client
- The catch-all response message is always generic — never leak internals

---

## 6. Defensive Coding — Verify Assumptions Explicitly

When receiving data from external sources (HTTP, messaging, database),
verify its shape before processing. Do not assume it matches expectations.

```java
// ❌ BAD: assumes message payload is valid
public void handlePolicyEvent(PolicyEvent event) {
    policyService.activate(event.getPolicyNumber());
}

// ✅ GOOD: verify before processing
public void handlePolicyEvent(PolicyEvent event) {
    Assert.notNull(event, "event must not be null");
    Assert.hasText(event.getPolicyNumber(), "event.policyNumber must not be blank");
    Assert.notNull(event.getEventType(), "event.eventType must not be null");

    policyService.activate(event.getPolicyNumber());
}
```

For data coming from repositories or external APIs — always consider:
- Can this return null?
- Can this collection be empty?
- Can this numeric value be zero or negative?
- Can this string be blank even if not null?

Verify each assumption explicitly rather than finding out at a NullPointerException
three stack frames later.

---

## Anti-Patterns

- **Silent catch** — `catch (Exception e) {}` — the worst kind of failure
- **Exception log spam** — logging at every layer, same exception logged 5 times
- **Generic RuntimeException** — `throw new RuntimeException("something went wrong")` — meaningless
- **Null returns for absence** — return `Optional` or throw a domain exception
- **Controller try/catch** — domain exception handling belongs in `@ControllerAdvice`
- **Leaking internals** — returning stack traces, class names, or SQL errors to the client
- **`orElse(null)` followed by null check** — defeats the purpose of Optional entirely
- **Ignoring Spring Assert in favor of manual if/throw** — verbose and inconsistent

---

## Quick Checklist

Before presenting any Java service or controller code:

- [ ] All method parameters validated with `Assert` before logic runs
- [ ] All `Optional` results handled with `orElseThrow()`, `orElse()`, or `ifPresent()` — never `.get()`
- [ ] No `catch` block that only logs without rethrowing or handling
- [ ] Domain errors use named custom exceptions, not raw `RuntimeException`
- [ ] No try/catch in controllers — `@ControllerAdvice` handles it
- [ ] Catch-all handler present in `@ControllerAdvice`
- [ ] No internal details (stack traces, SQL, class names) in HTTP error responses
- [ ] External/messaging inputs validated defensively before processing
