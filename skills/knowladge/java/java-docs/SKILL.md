---
name: java-javadoc
description: >
  Rules for writing Javadoc on all public methods and classes. Enforces
  meaningful descriptions, @param, @return, and @throws on every public
  method. Bans useless, redundant, and auto-generated Javadoc that adds
  no value over the signature itself.
---

Every public method gets a Javadoc comment. Every Javadoc comment must add
information that is not already obvious from the method signature alone.

## The Iron Law

```
1. Every public method has a Javadoc
2. Every @param, @return, and @throws tag is always present
3. Javadoc describes WHY and WHAT — not HOW
4. Never restate the method name as the description
```

---

## Structure

```java
/**
 * [One sentence summary — what this method does and why it exists.]
 *
 * [Optional: additional context, constraints, or behavior worth knowing.
 *  Include edge cases, preconditions, or non-obvious side effects here.]
 *
 * @param paramName  description of what this parameter represents and any constraints
 * @param paramName2 description of what this parameter represents and any constraints
 * @return           description of what is returned and what it represents
 * @throws ExceptionType when this exception is thrown and under what condition
 */
```

**Rules:**
- First sentence is a summary — it appears in generated Javadoc indexes, keep it concise
- Additional paragraph for non-obvious behavior, constraints, or side effects
- `@param` — always present for every parameter, even obvious ones
- `@return` — always present unless return type is `void`
- `@throws` — always present for every checked and meaningful unchecked exception thrown

---

## Good vs Bad Javadoc

### Summary line — describe intent, not the name

```java
// ❌ BAD: restates the method name
/**
 * Activates the policy.
 */
public Policy activate(String policyNumber) { ... }

// ❌ BAD: describes implementation, not intent
/**
 * Sets the policy status field to ACTIVE and calls save on the repository.
 */
public Policy activate(String policyNumber) { ... }

// ✅ GOOD: describes what and why
/**
 * Activates a policy, making it eligible for claims and renewals.
 *
 * A policy can only be activated from DRAFT or SUSPENDED status.
 * Activation triggers a notification to the policyholder.
 *
 * @param policyNumber the unique policy identifier, must not be blank
 * @return             the updated policy in ACTIVE status
 * @throws PolicyNotFoundException   if no policy exists with the given number
 * @throws PolicyActivationException if the policy is not in an activatable status
 */
public Policy activate(String policyNumber) { ... }
```

---

### @param — describe meaning and constraints, not the type

```java
// ❌ BAD: restates the type
/**
 * @param policyNumber String representing the policy number
 * @param premium      BigDecimal for the premium amount
 */

// ❌ BAD: too vague
/**
 * @param policyNumber the policy number
 * @param premium      the premium
 */

// ✅ GOOD: describes meaning and constraints
/**
 * @param policyNumber unique policy identifier in format POL-XXXXXX, must not be blank
 * @param premium      annual premium amount in EUR, must be greater than zero
 */
```

---

### @return — describe what the value represents

```java
// ❌ BAD: restates the return type
/**
 * @return Optional<Policy>
 */

// ❌ BAD: too vague
/**
 * @return the policy
 */

// ✅ GOOD: describes what it contains and when it is empty
/**
 * @return the policy matching the given number,
 *         or empty if no policy exists with that number
 */

// ✅ GOOD: for collections
/**
 * @return all active policies for the given client,
 *         or an empty list if none exist — never null
 */
```

Always state **never null** explicitly when the return value is guaranteed
non-null — it is a contract worth documenting.

---

### @throws — describe the condition, not just the type

```java
// ❌ BAD: restates the exception name
/**
 * @throws PolicyNotFoundException   PolicyNotFoundException
 * @throws IllegalArgumentException  IllegalArgumentException
 */

// ✅ GOOD: describes when it is thrown
/**
 * @throws PolicyNotFoundException   if no policy exists with the given policyNumber
 * @throws PolicyActivationException if the policy status does not allow activation
 * @throws IllegalArgumentException  if policyNumber is null or blank
 */
```

Document every exception the method explicitly throws.
For `IllegalArgumentException` from Spring Assert — always document it.

---

## Class-Level Javadoc

Every public class and interface gets a class-level Javadoc describing
its responsibility in the system.

```java
// ❌ BAD
/**
 * Policy service.
 */
@Service
public class PolicyService { ... }

// ✅ GOOD
/**
 * Application service responsible for policy lifecycle operations —
 * creation, activation, suspension, and cancellation.
 *
 * Orchestrates between the domain model, persistence layer, and
 * notification service. All business rule enforcement for policy
 * state transitions is delegated to the {@link Policy} domain object.
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class PolicyService { ... }
```

---

## Interface vs Implementation

When a method is declared on an interface and implemented in a class:
- Write the full Javadoc on the **interface method**
- Use `{@inheritDoc}` on the implementation — do not duplicate

```java
public interface PolicyRepository extends JpaRepository<Policy, Long> {

    /**
     * Finds all policies currently in ACTIVE status.
     *
     * @return all active policies, or an empty list if none exist — never null
     */
    List<Policy> findAllByStatus(PolicyStatus status);
}

// Implementation
@Repository
public class PolicyRepositoryImpl implements PolicyRepository {

    /** {@inheritDoc} */
    @Override
    public List<Policy> findAllByStatus(PolicyStatus status) { ... }
}
```

---

## Inline Tags Worth Using

| Tag | Use for |
|---|---|
| `{@link ClassName}` | Reference a related class or method |
| `{@link ClassName#method}` | Reference a specific method |
| `{@code value}` | Inline code snippet or literal value |
| `{@inheritDoc}` | Inherit Javadoc from interface or superclass |

```java
/**
 * Calculates the premium for a policy using the {@link PremiumCalculator}.
 *
 * Returns {@code BigDecimal.ZERO} if the policy has no coverage items.
 *
 * @param policy the policy to calculate premium for, must not be null
 * @return       the calculated premium, never null, never negative
 * @throws IllegalArgumentException if policy is null
 */
```

---

## What Not to Javadoc

Some things do not need Javadoc even on public methods — they add noise,
not value:

```java
// ❌ Javadoc that adds nothing
/**
 * Gets the policy number.
 *
 * @return the policy number
 */
public String getPolicyNumber() { return policyNumber; }
```

**Exception to the rule:** Simple getters and setters generated by Lombok
do not need Javadoc — Lombok signals intent through annotations.
Write Javadoc on the **field** instead if documentation is needed:

```java
@Data
public class Policy {

    /** Unique policy identifier in format POL-XXXXXX. */
    private String policyNumber;

    /** Annual premium amount in EUR. Always greater than zero. */
    private BigDecimal premium;
}
```

---

## Anti-Patterns

- **Redundant summary** — `Activates the policy` on a method called `activate()` — adds nothing
- **Implementation description** — describing how the method works internally, not what it does
- **Missing @throws** — a method throws `PolicyNotFoundException` but Javadoc doesn't mention it
- **Vague @param** — `@param id the id` — what kind of id? What format? What constraints?
- **Missing "never null" contract** — callers need to know if they can trust the return value
- **Copy-paste Javadoc** — same description on five different methods, clearly not read
- **Auto-generated stubs** — `/** TODO */` or `/** @param param1 param1 */` — worse than no Javadoc

---

## Quick Checklist

Before presenting any public Java method:

- [ ] Method has a Javadoc comment
- [ ] Summary line describes intent — not the method name, not the implementation
- [ ] Every parameter has a `@param` tag with meaning and constraints
- [ ] `@return` tag present unless void — includes "never null" when guaranteed
- [ ] Every thrown exception has a `@throws` tag describing the condition
- [ ] Class has a class-level Javadoc describing its responsibility
- [ ] Lombok-generated getters/setters documented on the field, not the method
- [ ] Interface methods carry full Javadoc — implementations use `{@inheritDoc}`
