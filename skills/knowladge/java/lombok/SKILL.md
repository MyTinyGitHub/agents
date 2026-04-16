---
name: java-lombok
description: >
  Rules for using Lombok in Java. Prioritizes @Data, @Builder, and other
  annotations over manual getters, setters, constructors, equals, hashCode,
  and toString. Use on any Java task involving POJOs, DTOs, entities, or
  any class that would otherwise require boilerplate generation.
---

Never write getters, setters, constructors, `equals()`, `hashCode()`, or
`toString()` by hand when Lombok can generate them. Manual boilerplate is
noise — it obscures the class's actual purpose and breaks when fields change.

## The Iron Law

```
NO manual getters. NO manual setters. NO manual constructors for data classes.
Lombok generates them. You declare intent with annotations.
```

---

## Annotation Decision Tree

```
Is this a pure data carrier (DTO, request, response)?
  → @Data + @Builder

Is this immutable (value object, read-only response)?
  → @Value + @Builder

Is this a JPA @Entity?
  → @Getter + @Setter only (never @Data on entities — see below)

Does it only need reading (no mutation)?
  → @Getter only

Do you need a builder but the class is otherwise manual?
  → @Builder alone
```

---

## @Data — The Default for Mutable Data Classes

`@Data` generates: `@Getter`, `@Setter`, `@RequiredArgsConstructor`,
`@EqualsAndHashCode`, `@ToString`.

```java
// ❌ BAD: manual boilerplate
public class PolicyRequest {
    private String policyNumber;
    private BigDecimal premium;
    private LocalDate startDate;

    public String getPolicyNumber() { return policyNumber; }
    public void setPolicyNumber(String policyNumber) { this.policyNumber = policyNumber; }
    public BigDecimal getPremium() { return premium; }
    public void setPremium(BigDecimal premium) { this.premium = premium; }
    // ... equals, hashCode, toString
}

// ✅ GOOD
@Data
public class PolicyRequest {
    private String policyNumber;
    private BigDecimal premium;
    private LocalDate startDate;
}
```

---

## @Builder — The Default for Object Construction

Always pair `@Builder` with `@Data` for classes that are constructed with
multiple optional fields. Eliminates telescoping constructors.

```java
// ❌ BAD: telescoping constructors
public PolicyRequest(String policyNumber) { ... }
public PolicyRequest(String policyNumber, BigDecimal premium) { ... }
public PolicyRequest(String policyNumber, BigDecimal premium, LocalDate startDate) { ... }

// ✅ GOOD
@Data
@Builder
public class PolicyRequest {
    private String policyNumber;
    private BigDecimal premium;
    private LocalDate startDate;
}

// Usage
PolicyRequest request = PolicyRequest.builder()
    .policyNumber("POL-001")
    .premium(new BigDecimal("1500.00"))
    .startDate(LocalDate.now())
    .build();
```

### @Builder.Default for default field values

```java
@Data
@Builder
public class PolicyRequest {
    private String policyNumber;

    @Builder.Default
    private PolicyStatus status = PolicyStatus.DRAFT;

    @Builder.Default
    private List<String> endorsements = new ArrayList<>();
}
```

Without `@Builder.Default`, builder-constructed instances will have `null`
for fields with initializers — a common Lombok gotcha.

---

## @Value — For Immutable Classes

`@Value` is the immutable variant of `@Data`.
Generates: all-args constructor, getters only (no setters), final fields,
`equals`, `hashCode`, `toString`.

```java
// ✅ GOOD: immutable value object
@Value
@Builder
public class PolicyId {
    String value;
}

// ✅ GOOD: immutable response
@Value
@Builder
public class PolicySummary {
    String policyNumber;
    BigDecimal premium;
    PolicyStatus status;
}
```

Prefer `@Value` over `@Data` when the object should not change after construction.
Prefer Java `record` over `@Value` for simple carriers with no builder need — see idiomatic-java skill.

---

## @Getter / @Setter — When You Need Partial Control

Use when `@Data` is too broad — for example when only reads are needed,
or when setters should be restricted to specific fields.

```java
// ✅ GOOD: read-only class
@Getter
public class AuditEntry {
    private final String actor;
    private final String action;
    private final Instant timestamp;
}

// ✅ GOOD: selective setter
@Getter
public class Policy {
    private final String policyNumber;  // never changes

    @Setter
    private PolicyStatus status;        // mutable
}
```

---

## @RequiredArgsConstructor — For Constructor Injection

Spring constructor injection without writing the constructor manually.

```java
// ❌ BAD: manual constructor
@Service
public class PolicyService {
    private final PolicyRepository policyRepository;
    private final PremiumCalculator premiumCalculator;

    public PolicyService(PolicyRepository policyRepository,
                         PremiumCalculator premiumCalculator) {
        this.policyRepository = policyRepository;
        this.premiumCalculator = premiumCalculator;
    }
}

// ✅ GOOD
@Slf4j
@Service
@RequiredArgsConstructor
public class PolicyService {
    private final PolicyRepository policyRepository;
    private final PremiumCalculator premiumCalculator;
}
```

All `final` fields are injected via the generated constructor.
This is the standard pattern for Spring services, repositories, and controllers.

---

## NEVER Use @Data on JPA Entities

`@Data` on a JPA `@Entity` causes serious problems:
- `@EqualsAndHashCode` uses all fields — breaks Hibernate proxy equality
- `@ToString` can trigger lazy loading and cause `LazyInitializationException`
- Bidirectional relationships cause infinite recursion in `toString`

```java
// ❌ BAD
@Data
@Entity
public class Policy { ... }

// ✅ GOOD: explicit and safe
@Getter
@Setter
@Entity
public class Policy {

    @Id
    @GeneratedValue
    private Long id;

    private String policyNumber;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Policy policy)) return false;
        return id != null && id.equals(policy.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

For JPA entities: `@Getter` + `@Setter` only. Write `equals`/`hashCode` manually
based on the business key or ID.

---

## @ToString — Exclude Sensitive or Recursive Fields

When using `@Data` or `@ToString` explicitly, exclude fields that should
not appear in logs:

```java
@Data
@Builder
public class UserRequest {
    private String username;

    @ToString.Exclude
    private String password;

    @ToString.Exclude
    private String creditCardNumber;
}
```

Also exclude fields that cause circular references in bidirectional relationships.

---

## Annotation Combinations — Quick Reference

| Use case | Annotations |
|---|---|
| Mutable DTO / request / response | `@Data @Builder` |
| Immutable value object | `@Value @Builder` |
| Simple data carrier (no builder) | `record` (prefer over `@Value`) |
| Spring service / controller | `@RequiredArgsConstructor` |
| JPA entity | `@Getter @Setter` + manual `equals`/`hashCode` |
| Read-only class | `@Getter` |
| Logging in any class | `@Slf4j` |

---

## Anti-Patterns

- **Manual getters/setters** when `@Data` or `@Getter`/`@Setter` applies — pure noise
- **`@Data` on `@Entity`** — breaks Hibernate, causes lazy loading exceptions
- **Missing `@Builder.Default`** on fields with initializers — silently produces nulls
- **`@AllArgsConstructor` for Spring injection** — use `@RequiredArgsConstructor` with `final` fields instead
- **`@ToString` on classes with sensitive fields** — always exclude passwords, tokens, PII
- **Mixing Lombok and manual boilerplate** — if Lombok is on the classpath, use it consistently

---

## Quick Checklist

Before presenting any Java data class:

- [ ] No manual getters or setters — `@Data`, `@Getter`, or `@Setter` instead
- [ ] No manual constructors for data classes — `@Builder` or `@RequiredArgsConstructor`
- [ ] No `@Data` on JPA `@Entity` classes
- [ ] `@Builder.Default` present on fields with non-null initializers
- [ ] Sensitive fields excluded from `@ToString`
- [ ] Spring services use `@RequiredArgsConstructor` with `final` fields
