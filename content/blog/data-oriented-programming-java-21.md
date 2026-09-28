---
title: "Data-oriented programming in Java 21: modelling domains with records, sealed interfaces and pattern matching"
date: 2026-09-28T00:00:00Z
description: "A step-by-step tutorial on modelling a payment domain with records, sealed interfaces and exhaustive switch in Java 21, replacing the Visitor pattern and wiring it into Jackson, JPA and JUnit 5."
---
# Data-oriented programming in Java 21: modelling domains with records, sealed interfaces and pattern matching

## 1. Overview

For years I modelled domain states in Java with one of two tools: an `enum` plus a lot of nullable fields, or a class hierarchy plus the Visitor pattern. Both work, but both hide mistakes. The compiler cannot tell me that a `CAPTURED` payment has no capture date, or that I forgot to handle a new state in one of twenty `if` chains.

Java 21 gives me a better option. With **records**, **sealed interfaces** and **pattern matching for `switch`**, I can describe the data first and let the compiler check that every piece of code handles every case. This style is often called **data-oriented programming**.

In this post I will build a small payment domain step by step:

1. Model values with records and validate them in the compact constructor.
2. Close the set of states with a sealed interface.
3. Add behaviour with exhaustive `switch`, record patterns and guards.
4. Replace a Visitor with a plain `switch`.
5. Write state transitions as pure functions.
6. Serialize the model with Jackson in Spring Boot.
7. Persist it with JPA without breaking the model.
8. Test it with JUnit 5.

All the code compiles with `javac --release 21`.

## 2. Problem

This is a typical "before" model:

```java
public class Payment {

    private PaymentStatus status;   // PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED
    private BigDecimal amount;
    private String authCode;        // only when AUTHORIZED
    private Instant expiresAt;      // only when AUTHORIZED
    private Instant capturedAt;     // only when CAPTURED
    private String failureReason;   // only when FAILED
    private Instant refundedAt;     // only when REFUNDED

    // getters and setters for everything
}
```

The problems are easy to see once you list them:

- **Invalid states are possible.** Nothing stops `status = CAPTURED` with `capturedAt = null`.
- **Knowledge is spread out.** The comments are the only place that says which field belongs to which state.
- **New states are silent.** If I add `DISPUTED`, every `if (status == ...)` in the code base still compiles.
- **Null checks everywhere.** Each consumer must remember which fields can be `null` in which state.

## 3. The building blocks

Data-oriented programming in Java uses three features. All of them are final in Java 21, so no preview flags are needed:

| Feature | What it gives me | Final since |
|---|---|---|
| Records (JEP 395) | Immutable data carriers with `equals`, `hashCode`, `toString` | Java 16 |
| Sealed classes and interfaces (JEP 409) | A closed, known set of subtypes | Java 17 |
| Record patterns (JEP 440) | Deconstruct a record inside `instanceof` / `switch` | Java 21 |
| Pattern matching for `switch` (JEP 441) | Type patterns, guards (`when`), exhaustiveness checks | Java 21 |

The idea is simple:

- **Data** is modelled as records. They are plain, immutable and transparent.
- **Choices** ("a payment is *one of* these states") are modelled as a sealed interface.
- **Behaviour** lives in functions that `switch` over the data. The compiler checks that the `switch` covers every case.

## 4. Step 1: model values with records

I start with the smallest value in the domain: money. A record gives me immutability and value equality for free. The **compact constructor** is the place to validate and normalize input:

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount, "amount");
        Objects.requireNonNull(currency, "currency");
        if (amount.signum() < 0) {
            throw new IllegalArgumentException("amount must be >= 0, was " + amount);
        }
        amount = amount.setScale(currency.getDefaultFractionDigits());
    }

    public static Money eur(String amount) {
        return new Money(new BigDecimal(amount), Currency.getInstance("EUR"));
    }

    public boolean isGreaterThan(Money other) {
        requireSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch: " + currency + " vs " + other.currency);
        }
    }
}
```

A few details are worth pointing out:

- In a compact constructor I do not write `this.amount = amount`. I reassign the **parameter**, and Java assigns the fields at the end.
- `setScale` without a rounding mode throws `ArithmeticException` if rounding would be needed. For money that is what I want: `Money.eur("10.005")` fails fast instead of losing a cent.
- Because `Money.eur("10")` and `Money.eur("10.00")` are normalized to the same scale, `equals` works as expected.

**Rule of thumb:** if an object is "just data" and two instances with the same values are the same thing, make it a record.

## 5. Step 2: close the states with a sealed interface

Next I describe what a payment state *is*. Each state is a record with **only the fields that make sense for that state**. The sealed interface lists every allowed state:

```java
public sealed interface PaymentState
        permits PaymentState.Pending, PaymentState.Authorized, PaymentState.Captured,
                PaymentState.Failed, PaymentState.Refunded {

    record Pending(Money amount) implements PaymentState {
        public Pending {
            Objects.requireNonNull(amount, "amount");
        }
    }

    record Authorized(Money amount, String authCode, Instant expiresAt) implements PaymentState {
        public Authorized {
            Objects.requireNonNull(amount, "amount");
            if (authCode == null || authCode.isBlank()) {
                throw new IllegalArgumentException("authCode is required");
            }
            Objects.requireNonNull(expiresAt, "expiresAt");
        }
    }

    record Captured(Money amount, Instant capturedAt) implements PaymentState {
        public Captured {
            Objects.requireNonNull(amount, "amount");
            Objects.requireNonNull(capturedAt, "capturedAt");
        }
    }

    record Failed(String reason, boolean retryable) implements PaymentState {
        public Failed {
            Objects.requireNonNull(reason, "reason");
        }
    }

    record Refunded(Money amount, Instant refundedAt) implements PaymentState {
        public Refunded {
            Objects.requireNonNull(amount, "amount");
            Objects.requireNonNull(refundedAt, "refundedAt");
        }
    }
}
```

Compare this with the class from section 2:

- A `Captured` payment **cannot** exist without `capturedAt`. The constructor refuses it.
- A `Failed` payment has no `authCode` field at all, so nobody can read a stale one.
- The list of states is in one place, and the compiler knows it is complete.

Some notes on sealed types:

- Records are implicitly `final`, so they are valid permitted subtypes without extra modifiers.
- When all subtypes are in the same file (as nested types here), the `permits` clause is optional. I keep it because it reads like documentation.
- If the subtypes live in separate files, they must be in the **same package** (or the same named module) as the sealed interface.

## 6. Step 3: add behaviour with an exhaustive switch

Now I can write behaviour as functions over the data. A `switch` over a sealed type is **exhaustive**: if all permitted subtypes are covered, no `default` is needed.

### 6.1. Type patterns

```java
public static String describe(PaymentState state) {
    return switch (state) {
        case Pending p -> "Waiting for authorization of " + p.amount().amount();
        case Authorized a -> "Authorized with code " + a.authCode();
        case Captured c -> "Captured at " + c.capturedAt();
        case Failed f -> "Failed: " + f.reason();
        case Refunded r -> "Refunded " + r.amount().amount();
    };
}
```

If I delete the `Refunded` line, the build fails:

```text
PaymentViews.java:11: error: the switch expression does not cover all possible input values
        return switch (state) {
               ^
```

This is the main benefit. When I add a new state (say `Disputed`) to the `permits` list, the compiler gives me a list of every place I have to update.

### 6.2. Record patterns and guards

Record patterns let me **deconstruct** a record directly in the `case`. Patterns can be nested, and `when` adds a condition (a *guard*):

```java
public static String label(PaymentState state) {
    return switch (state) {
        case Pending(Money(var amount, var currency)) -> "PENDING " + amount + " " + currency;
        case Authorized(var amount, var code, var expiresAt) -> "AUTHORIZED " + code;
        case Captured(var amount, var at) -> "CAPTURED";
        case Failed(var reason, var retryable) when retryable -> "FAILED (retry) " + reason;
        case Failed(var reason, var retryable) -> "FAILED " + reason;
        case Refunded(var amount, var at) -> "REFUNDED";
    };
}
```

Things to know:

- **Order matters.** Cases are checked from top to bottom. The guarded `Failed ... when retryable` must come before the unguarded `Failed`. If you put it after, the compiler reports that the case is dominated.
- A guarded case **does not count** for exhaustiveness. I still need the unguarded `Failed` case.
- In Java 21 you must name every component. The unnamed pattern `_` (for example `case Captured(var amount, _)`) is a preview in 21 and final in Java 22 (JEP 456).

### 6.3. Avoid `default`

It is tempting to write this:

```java
return switch (state) {
    case Refunded r -> true;
    default -> false;          // compiles today, silently wrong tomorrow
};
```

A `default` branch turns off the exhaustiveness check. When a new state appears, it falls into `default` without a warning. I prefer listing every case, even when several return the same value:

```java
public static boolean isFinal(PaymentState state) {
    return switch (state) {
        case Pending p -> false;
        case Authorized a -> false;
        case Failed f -> !f.retryable();
        case Captured c -> false;
        case Refunded r -> true;
    };
}
```

## 7. Step 4: replace the Visitor pattern

With a class hierarchy, the classic way to add operations without `instanceof` is the Visitor pattern:

```java
interface PaymentStateVisitor<R> {
    R visitPending(Pending p);
    R visitAuthorized(Authorized a);
    R visitCaptured(Captured c);
    R visitFailed(Failed f);
    R visitRefunded(Refunded r);
}

// each subtype
public <R> R accept(PaymentStateVisitor<R> visitor) {
    return visitor.visitPending(this);
}

// each operation
String text = state.accept(new PaymentStateVisitor<>() {
    public String visitPending(Pending p) { return "..."; }
    // four more methods
});
```

The Visitor gives two guarantees: every operation handles every type, and the types do not need to know about the operations. A `switch` over a sealed interface gives **the same two guarantees** with no `accept` method, no visitor interface and no double dispatch. The `describe` method in section 6.1 is the complete replacement.

When is a Visitor still useful? When the hierarchy is **open** (third parties can add types) you cannot seal it, so the compiler cannot check exhaustiveness. For a domain model that I own, the sealed interface is the simpler choice.

## 8. Step 5: write transitions as pure functions

State changes are also functions: they take a state (and some input) and return a **new** state. Nothing is mutated. I inject a `Clock` so the code is easy to test:

```java
public final class PaymentTransitions {

    private final Clock clock;

    public PaymentTransitions(Clock clock) {
        this.clock = clock;
    }

    public PaymentState capture(PaymentState state) {
        Instant now = clock.instant();
        return switch (state) {
            case Authorized(var amount, var code, var expiresAt) when expiresAt.isBefore(now) ->
                    new Failed("authorization " + code + " expired", true);
            case Authorized(var amount, var code, var expiresAt) ->
                    new Captured(amount, now);
            case Pending p -> throw illegal("capture", p);
            case Captured c -> throw illegal("capture", c);
            case Failed f -> throw illegal("capture", f);
            case Refunded r -> throw illegal("capture", r);
        };
    }

    public PaymentState refund(PaymentState state, Money requested) {
        return switch (state) {
            case Captured(var captured, var at) when requested.isGreaterThan(captured) ->
                    throw new IllegalArgumentException("cannot refund more than captured");
            case Captured c -> new Refunded(requested, clock.instant());
            case Pending p -> throw illegal("refund", p);
            case Authorized a -> throw illegal("refund", a);
            case Failed f -> throw illegal("refund", f);
            case Refunded r -> throw illegal("refund", r);
        };
    }

    private static IllegalStateException illegal(String action, PaymentState state) {
        return new IllegalStateException(
                "cannot " + action + " a payment in state " + state.getClass().getSimpleName());
    }
}
```

The whole state machine is now visible in two methods. Each allowed transition is one line, each forbidden transition is explicit, and a new state forces me to decide what `capture` and `refund` should do with it.

## 9. Step 6: JSON with Jackson in Spring Boot

Jackson supports records since version 2.12, and Spring Boot 3.x ships a recent Jackson with `JavaTimeModule` already registered. For the sealed interface I still need to tell Jackson how to pick the subtype. I add a type property:

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "status")
@JsonSubTypes({
        @JsonSubTypes.Type(value = PaymentState.Pending.class, name = "PENDING"),
        @JsonSubTypes.Type(value = PaymentState.Authorized.class, name = "AUTHORIZED"),
        @JsonSubTypes.Type(value = PaymentState.Captured.class, name = "CAPTURED"),
        @JsonSubTypes.Type(value = PaymentState.Failed.class, name = "FAILED"),
        @JsonSubTypes.Type(value = PaymentState.Refunded.class, name = "REFUNDED")
})
public sealed interface PaymentState permits /* ... */ {
    // records as before
}
```

A `Captured` state is then serialized like this:

```json
{
  "status": "CAPTURED",
  "amount": { "amount": 49.90, "currency": "EUR" },
  "capturedAt": "2026-09-28T10:00:00Z"
}
```

And a controller can return the domain type directly:

```java
@RestController
@RequestMapping("/payments")
class PaymentController {

    private final PaymentService payments;

    PaymentController(PaymentService payments) {
        this.payments = payments;
    }

    @GetMapping("/{id}/state")
    PaymentState state(@PathVariable UUID id) {
        return payments.currentState(id);
    }
}
```

Two practical tips:

- The subtype list in `@JsonSubTypes` must be kept in sync with `permits`. A test that serializes and deserializes one instance of every subtype (section 11) catches a missing entry.
- Deserialization runs the compact constructor, so **invalid JSON is rejected** with the same validation as in the code. A `CAPTURED` payload without `capturedAt` fails with an exception instead of creating a broken object.

If you prefer to keep Jackson annotations out of the domain package, put them on a **mix-in** class and register it with a `Jackson2ObjectMapperBuilderCustomizer`.

## 10. Step 7: persistence with JPA

Records are not a good fit for JPA **entities**. Entities need a no-argument constructor, mutable fields and proxies, and records have none of those. My approach is to keep the entity as a boring persistence shape and **map** to and from the sealed type at the edge:

```java
@Entity
@Table(name = "payment")
class PaymentEntity {

    @Id
    UUID id;

    @Enumerated(EnumType.STRING)
    Status status;                 // PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED

    BigDecimal amount;
    String currency;
    String authCode;
    Instant expiresAt;
    Instant capturedAt;
    Instant refundedAt;
    String failureReason;
    Boolean retryable;

    enum Status { PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED }
}
```

The mapper is again an exhaustive `switch` in one direction, and a `switch` over the enum in the other:

```java
final class PaymentStateMapper {

    static PaymentState toDomain(PaymentEntity e) {
        Money money = e.amount == null ? null : new Money(e.amount, Currency.getInstance(e.currency));
        return switch (e.status) {
            case PENDING -> new PaymentState.Pending(money);
            case AUTHORIZED -> new PaymentState.Authorized(money, e.authCode, e.expiresAt);
            case CAPTURED -> new PaymentState.Captured(money, e.capturedAt);
            case FAILED -> new PaymentState.Failed(e.failureReason, Boolean.TRUE.equals(e.retryable));
            case REFUNDED -> new PaymentState.Refunded(money, e.refundedAt);
        };
    }

    static void apply(PaymentState state, PaymentEntity e) {
        clearStateFields(e);
        switch (state) {
            case PaymentState.Pending(var money) -> {
                e.status = PaymentEntity.Status.PENDING;
                setMoney(e, money);
            }
            case PaymentState.Authorized(var money, var code, var expiresAt) -> {
                e.status = PaymentEntity.Status.AUTHORIZED;
                setMoney(e, money);
                e.authCode = code;
                e.expiresAt = expiresAt;
            }
            case PaymentState.Captured(var money, var at) -> {
                e.status = PaymentEntity.Status.CAPTURED;
                setMoney(e, money);
                e.capturedAt = at;
            }
            case PaymentState.Failed(var reason, var retryable) -> {
                e.status = PaymentEntity.Status.FAILED;
                e.failureReason = reason;
                e.retryable = retryable;
            }
            case PaymentState.Refunded(var money, var at) -> {
                e.status = PaymentEntity.Status.REFUNDED;
                setMoney(e, money);
                e.refundedAt = at;
            }
        }
    }

    private static void setMoney(PaymentEntity e, Money money) {
        e.amount = money.amount();
        e.currency = money.currency().getCurrencyCode();
    }

    private static void clearStateFields(PaymentEntity e) {
        e.authCode = null;
        e.expiresAt = null;
        e.capturedAt = null;
        e.refundedAt = null;
        e.failureReason = null;
        e.retryable = null;
    }
}
```

Why is this better than using the entity everywhere?

- The nullable columns stay **inside the persistence layer**. Services and controllers only see `PaymentState`.
- Reading a broken row (for example `CAPTURED` with a `null` `capturedAt`) fails in `toDomain`, close to the database, instead of somewhere random later.
- A `switch` statement over a sealed type with patterns is also checked for exhaustiveness, so `apply` breaks the build when a new state appears.

For small value objects like `Money`, Hibernate 6.2+ also supports records as `@Embeddable` types, which can remove some of the manual mapping.

## 11. Step 8: test it with JUnit 5

Because the model is immutable and the transitions are pure functions, the tests are short. A fixed `Clock` makes time deterministic:

```java
class PaymentTransitionsTest {

    private static final Instant NOW = Instant.parse("2026-09-28T10:00:00Z");
    private final PaymentTransitions transitions =
            new PaymentTransitions(Clock.fixed(NOW, ZoneOffset.UTC));

    @Test
    void capturesAValidAuthorization() {
        var authorized = new Authorized(Money.eur("49.90"), "A-123", NOW.plus(Duration.ofDays(1)));

        var result = transitions.capture(authorized);

        assertEquals(new Captured(Money.eur("49.90"), NOW), result);
    }

    @Test
    void failsWhenTheAuthorizationExpired() {
        var authorized = new Authorized(Money.eur("49.90"), "A-123", NOW.minusSeconds(1));

        var result = transitions.capture(authorized);

        assertInstanceOf(Failed.class, result);
        assertTrue(((Failed) result).retryable());
    }

    @Test
    void rejectsRefundLargerThanCapture() {
        var captured = new Captured(Money.eur("10"), NOW);

        assertThrows(IllegalArgumentException.class,
                () -> transitions.refund(captured, Money.eur("10.01")));
    }

    @ParameterizedTest
    @MethodSource("statesThatCannotBeCaptured")
    void rejectsCaptureFromOtherStates(PaymentState state) {
        assertThrows(IllegalStateException.class, () -> transitions.capture(state));
    }

    static Stream<PaymentState> statesThatCannotBeCaptured() {
        return Stream.of(
                new Pending(Money.eur("1")),
                new Captured(Money.eur("1"), NOW),
                new Failed("card declined", false),
                new Refunded(Money.eur("1"), NOW));
    }
}
```

Two extra tests I always add:

- **Validation tests** for each compact constructor (`assertThrows` on `new Captured(money, null)`).
- **A JSON round trip** for one instance of every permitted subtype. I get the list with `PaymentState.class.getPermittedSubclasses()` and assert that each subtype has a sample, so a new state without a sample fails the test.

Note that record `equals` makes `assertEquals` compare values, so I do not need custom matchers. If you use the Object Mother pattern (for example with my library [Matriarch](/blog/matriarch-object-mother-java/)), records are a very natural target for generated fixtures.

## 12. Key decisions and pitfalls

- **Records are shallowly immutable.** A record with a `List` field still exposes a mutable list. Copy it in the compact constructor with `List.copyOf(items)`.
- **No `default` in a `switch` over a sealed type.** It hides new cases. List every subtype.
- **Keep behaviour outside the records when it depends on context.** Small, local rules (like `Money.isGreaterThan`) can live in the record. Rules that need a clock, a repository or configuration belong in a service like `PaymentTransitions`.
- **Do not make records JPA entities.** Map at the edge instead.
- **Sealed hierarchies are for closed sets.** If other teams or plugins must add new types, use a normal interface and accept that the compiler cannot check exhaustiveness.
- **Watch the Java version.** Record patterns and pattern matching for `switch` are final in 21. Unnamed patterns (`_`) need Java 22 or `--enable-preview` on 21.

## 13. Checklist

When I model a new domain concept in this style, I go through this list:

1. Identify the **values** (money, identifiers, ranges) and make them records with validation in the compact constructor.
2. Identify the **choices** ("X is one of A, B or C") and make them a sealed interface with one record per case.
3. Give each case **only the fields that belong to it**. No nullable "maybe" fields.
4. Write behaviour as **functions with an exhaustive `switch`**. No `default`.
5. Write state changes as **pure functions** that return a new state. Inject a `Clock` for time.
6. Add `@JsonTypeInfo` / `@JsonSubTypes` (or a mix-in) and keep the list in sync with `permits`.
7. Map to and from JPA entities in **one mapper**. Keep entities out of the domain.
8. Test constructors, transitions and a JSON round trip for every subtype.

## 14. Conclusion

Records, sealed interfaces and pattern matching are not just shorter syntax. Together they let me make invalid states impossible to build, keep the list of cases in one place, and let the compiler find every piece of code that must change when the domain changes.

For me the biggest win is the exhaustive `switch`. It replaces the Visitor pattern, the `if (status == ...)` chains and many of the null checks, and it turns "did we handle the new state everywhere?" from a code review question into a compile error.
