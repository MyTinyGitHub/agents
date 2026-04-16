---
name: java-logging
description: >
  Rules for logging in Java using SLF4J with Lombok @Slf4j. Use on any Java
  task that involves logging, debugging output, or error handling. Enforces
  structured, leveled logging. Bans System.out and printStackTrace entirely.
---

Never use `System.out.print`, `System.err.print`, or `e.printStackTrace()`.
All logging goes through SLF4J via Lombok `@Slf4j`.

## The Iron Law

```
NO System.out. NO System.err. NO printStackTrace().
All output goes through log.
```

---

## Setup

Add `@Slf4j` at the class level. Lombok generates the `log` field automatically.

```java
// ✅ GOOD
@Slf4j
@Service
public class PolicyService {

    public void activate(String policyNumber) {
        log.info("Activating policy: {}", policyNumber);
    }
}
```

```java
// ❌ BAD: manual logger declaration
public class PolicyService {
    private static final Logger log = LoggerFactory.getLogger(PolicyService.class);
}
```

```java
// ❌ BAD: System.out
public class PolicyService {
    public void activate(String policyNumber) {
        System.out.println("Activating policy: " + policyNumber);
    }
}
```

---

## Log Levels — Use the Right One

| Level | When to use |
|---|---|
| `log.error()` | Unexpected failures — exceptions, unrecoverable states |
| `log.warn()` | Expected but notable problems — retries, degraded behavior, deprecated usage |
| `log.info()` | Significant business events — policy created, payment processed, job completed |
| `log.debug()` | Internal flow details useful during development — method entry, intermediate values |
| `log.trace()` | Fine-grained detail — loop iterations, low-level I/O (rarely needed) |

**Rules:**
- `info` is for business events, not internal flow — do not log every method call at info
- `debug` is for the developer, not the operator — assume it is off in production
- `error` must always include the exception object — never just the message
- `warn` is not a softer `error` — only use it if the system can continue normally

---

## Always Use Parameterized Logging

Never concatenate strings in log statements.
SLF4J uses `{}` placeholders — the string is only built if the level is enabled.

```java
// ❌ BAD: string concatenation — always evaluated, even if debug is off
log.debug("Processing policy " + policyNumber + " for client " + clientId);

// ✅ GOOD: parameterized — evaluated only if debug is enabled
log.debug("Processing policy {} for client {}", policyNumber, clientId);
```

---

## Logging Exceptions — Always Pass the Throwable

```java
// ❌ BAD: logs message only, loses stack trace
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    log.error("Failed to save policy: {}", e.getMessage());
}

// ❌ BAD: printStackTrace bypasses logging framework
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    e.printStackTrace();
}

// ✅ GOOD: pass the throwable as the last argument — SLF4J logs the full stack trace
try {
    policyRepository.save(policy);
} catch (DataAccessException e) {
    log.error("Failed to save policy {}", policyNumber, e);
}
```

The throwable must always be the **last argument** — no `{}` placeholder for it.

---

## What to Log and Where

### Service layer — business events
```java
@Slf4j
@Service
public class PolicyService {

    public Policy activate(String policyNumber) {
        log.info("Activating policy {}", policyNumber);

        Policy policy = policyRepository.findByNumber(policyNumber)
            .orElseThrow(() -> {
                log.warn("Policy not found for activation: {}", policyNumber);
                return new PolicyNotFoundException(policyNumber);
            });

        policy.activate();
        policyRepository.save(policy);

        log.info("Policy {} activated successfully", policyNumber);
        return policy;
    }
}
```

### Repository / infrastructure layer — errors only
Log at `debug` for queries, `error` for failures.
Do not duplicate what the service already logs.

### Controller layer — do not log here
The service owns the business event. The controller delegates.
HTTP access logging is handled by the framework (Spring, Tomcat) — do not replicate it.

---

## Guard debug/trace Blocks for Expensive Calls

If building the log message involves a non-trivial computation:

```java
// ❌ BAD: toString() called even if debug is off
log.debug("Policy state: {}", policy.toDetailedString());

// ✅ GOOD: guard with isDebugEnabled()
if (log.isDebugEnabled()) {
    log.debug("Policy state: {}", policy.toDetailedString());
}
```

---

## Anti-Patterns

- **`System.out.println`** — never. Not even temporarily. Use `log.debug()`.
- **`e.printStackTrace()`** — never. Pass the throwable to `log.error()`.
- **Swallowed exceptions** — `catch (Exception e) { log.error("error"); }` loses the stack trace.
- **Logging and rethrowing the same exception twice** — log once, at the layer that handles it.
- **Sensitive data in logs** — never log passwords, tokens, full card numbers, or personal data.
- **Log spam at info** — logging every method entry/exit at `info` makes logs unreadable in production.
- **String concatenation in log args** — always use `{}` placeholders.

---

## Quick Checklist

Before presenting any Java code with output or error handling:

- [ ] Class has `@Slf4j` annotation
- [ ] No `System.out`, `System.err`, or `printStackTrace()` anywhere
- [ ] All placeholders use `{}` — no string concatenation
- [ ] Every `catch` block that logs passes the throwable as the last argument
- [ ] Log level matches the significance of the event
- [ ] No sensitive data in any log statement
