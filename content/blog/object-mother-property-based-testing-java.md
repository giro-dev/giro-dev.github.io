---
title: "Object Mother meets property-based testing in Java"
date: 2026-07-21T00:00:00Z
description: "A hands-on tutorial combining the Object Mother pattern with jqwik property-based testing to generate valid yet varied Java test data."
draft: true
---

## 1. Overview

Most Java test suites are built on *example-based* tests: we pick a few concrete inputs, run the code, and assert on the result. This works, but it only exercises the handful of values we happened to think of. Bugs love to live in the gaps between those examples.

*Property-based testing* takes a different angle. Instead of asserting on fixed inputs, we describe a **property** that must hold for *any* valid input, and the framework generates hundreds of cases trying to break it. The catch: naive random generators tend to produce structurally invalid objects, and we end up testing our validation code instead of our business logic.

In this tutorial we'll combine two ideas that fit together nicely:

- The **Object Mother** pattern (via the [Matriarch](/blog/matriarch-object-mother-java/) library) to guarantee that every generated object is *valid*.
- **jqwik** to *vary* the parts of the object that actually stress our properties.

We'll build a small pricing domain, write example-based tests first, then convert them into property-based tests and watch jqwik find an edge case we missed.

## 2. Maven Dependencies

We'll use JUnit 5, jqwik for property-based testing, AssertJ for fluent assertions, and Matriarch for the Object Mother:

```xml
<dependencies>
    <dependency>
        <groupId>net.jqwik</groupId>
        <artifactId>jqwik</artifactId>
        <version>1.9.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.26.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>dev.giro</groupId>
        <artifactId>matriarch</artifactId>
        <version>1.0.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

jqwik brings its own JUnit 5 `TestEngine`, so no extra Surefire configuration is needed to run `@Property` methods alongside regular `@Test` methods.

## 3. The Domain Under Test

Let's model a tiny order-and-pricing domain. An `Order` has a currency and a list of `OrderLine`s; each line has a unit price and a quantity:

```java
public record OrderLine(String sku, Money unitPrice, int quantity) {

    public OrderLine {
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }

    public Money lineTotal() {
        return unitPrice.multiply(quantity);
    }
}

public record Order(String currency, List<OrderLine> lines) {

    public Money total() {
        return lines.stream()
                .map(OrderLine::lineTotal)
                .reduce(Money.zero(currency), Money::add);
    }
}
```

`Money` is a small value object wrapping a `BigDecimal` amount and a currency code, with `add`, `multiply` and comparison helpers. The important invariant is that every `Money` in an `Order` shares the same currency.

The code we want to test is a percentage discount policy:

```java
public class PricingService {

    private final int discountPercentage;

    public PricingService(int discountPercentage) {
        this.discountPercentage = discountPercentage;
    }

    public Money discountFor(Order order) {
        return order.total().percentage(discountPercentage);
    }
}
```

The property we care about is simple to state in words: **a discount must never be negative and never exceed the order total.** That's exactly the kind of invariant property-based testing is good at.

## 4. Starting With an Example-Based Test

Before reaching for properties, let's write the test the way we usually would. This is also where the Object Mother earns its keep. We define a `Mother` once so tests don't drown in construction boilerplate:

```java
public final class OrderMother {

    private String currency = "EUR";
    private final List<OrderLine> lines = new ArrayList<>();

    public static OrderMother anOrder() {
        return new OrderMother();
    }

    public OrderMother inCurrency(String currency) {
        this.currency = currency;
        return this;
    }

    public OrderMother withLine(String sku, String unitPrice, int quantity) {
        lines.add(new OrderLine(sku, Money.of(unitPrice, currency), quantity));
        return this;
    }

    public Order build() {
        if (lines.isEmpty()) {
            withLine("SKU-DEFAULT", "10.00", 1);
        }
        return new Order(currency, List.copyOf(lines));
    }
}
```

The Mother owns the defaults and the invariants: it always produces at least one line, and every `Money` uses the order's currency. Now a classic example test reads cleanly:

```java
class PricingServiceExampleTest {

    private final PricingService pricing = new PricingService(10);

    @Test
    void givenTenPercentPolicy_whenDiscounting_thenReturnsTenPercentOfTotal() {
        Order order = OrderMother.anOrder()
                .withLine("SKU-1", "100.00", 2)   // 200.00
                .withLine("SKU-2", "50.00", 1)    //  50.00
                .build();                         // total = 250.00

        Money discount = pricing.discountFor(order);

        assertThat(discount).isEqualTo(Money.of("25.00", "EUR"));
    }
}
```

This test is correct and readable, but it only proves the policy works for *one* order totalling 250.00. It says nothing about empty-ish orders, huge quantities, or rounding at odd totals.

## 5. Writing the First Property

Let's express the invariant instead of a single example. In jqwik, a test method is annotated with `@Property`, and its parameters are annotated with `@ForAll`. A `@Provide` method supplies an `Arbitrary` — jqwik's generator — for a given type.

Here's the key move: the `Arbitrary` delegates object construction to the Object Mother, so every generated `Order` is valid by construction. jqwik only decides *how many lines* and *what quantities* to use:

```java
class PricingServicePropertyTest {

    private final PricingService pricing = new PricingService(10);

    @Property
    void discountIsBetweenZeroAndTotal(@ForAll("orders") Order order) {
        Money discount = pricing.discountFor(order);

        assertThat(discount.isNegative()).isFalse();
        assertThat(discount).isLessThanOrEqualTo(order.total());
    }

    @Provide
    Arbitrary<Order> orders() {
        Arbitrary<Integer> lineCount = Arbitraries.integers().between(1, 20);

        return lineCount.flatMap(count -> {
            Arbitrary<OrderLine> line = lines();
            return line.list().ofSize(count)
                    .map(generated -> OrderMother.anOrder()
                            .withLines(generated)   // Mother copies lines, keeps currency
                            .build());
        });
    }

    private Arbitrary<OrderLine> lines() {
        Arbitrary<String> sku = Arbitraries.strings().alpha().ofLength(6).map(s -> "SKU-" + s);
        Arbitrary<String> price = Arbitraries.bigDecimals()
                .between(BigDecimal.ZERO, new BigDecimal("1000"))
                .ofScale(2)
                .map(BigDecimal::toPlainString);
        Arbitrary<Integer> quantity = Arbitraries.integers().between(1, 100);

        return Combinators.combine(sku, price, quantity)
                .as((s, p, q) -> new OrderLine(s, Money.of(p, "EUR"), q));
    }
}
```

Notice the division of responsibility:

- **The Mother owns the invariants.** Currency consistency and "at least one line" never leak into the generator.
- **jqwik owns the variation.** It sweeps line counts (1–20), quantities (1–100) and prices (0–1000) — exactly the dimensions that stress the discount calculation.

By default jqwik runs this property with 1000 generated samples. If it can't break the property, the test passes just like any other JUnit test.

## 6. Letting jqwik Find the Bug

Now suppose the discount policy accepts *any* integer percentage, and somewhere a configuration allows values above 100. Let's parametrise the percentage with `@ForAll` too, over a slightly-too-wide range:

```java
@Property
void discountNeverExceedsTotal(
        @ForAll("orders") Order order,
        @ForAll @IntRange(min = 0, max = 150) int percentage) {

    Money discount = new PricingService(percentage).discountFor(order);

    assertThat(discount).isLessThanOrEqualTo(order.total());
}
```

jqwik quickly finds a counterexample and, crucially, **shrinks** it to the smallest failing input before reporting:

```
org.opentest4j.AssertionFailedError:
  Property [discountNeverExceedsTotal] falsified with sample
    order = Order[currency=EUR, lines=[OrderLine[sku=SKU-AAAAAA, unitPrice=0.01, quantity=1]]]
    percentage = 101

  Original sample: order with 14 lines, percentage = 137
  Shrunk sample:   order with 1 line of 0.01, percentage = 101

  tries = 43 | checks = 43 | seed = 8134572290117684019
```

Two things make this actionable:

1. **Shrinking** reduced a messy 14-line order down to a single line at the boundary (`percentage = 101`), pointing straight at the real problem: percentages above 100 are invalid input the policy should reject.
2. **The reported `seed`** makes the failure reproducible. Because the Object Mother uses that seed for its own data generation, we can replay the *exact* object — no "works on my machine" flakiness.

We can pin the seed while debugging to always regenerate the same case:

```java
@Property(seed = "8134572290117684019")
void discountNeverExceedsTotal(/* ... */) { /* ... */ }
```

The fix is a domain fix, not a test hack: `PricingService` should reject percentages outside `[0, 100]`. Once corrected, we narrow the generator back to a valid range and the property holds again.

## 7. Combining Example and Property Tests

Property-based tests don't replace example-based ones — they complement them:

- Keep **example tests** for documentation and for specific, business-meaningful scenarios ("a 10% discount on 250.00 is 25.00"). They read like specifications.
- Use **property tests** for invariants and policies that must hold universally ("discount is always within `[0, total]`", "serialise-then-deserialise round-trips", "adding a line never decreases the total").

Both styles share the same Object Mother, so there's a single source of truth for what a valid `Order` looks like. When the domain changes, we update the Mother once and both test styles keep compiling.

## 8. Conclusion

In this article we combined the **Object Mother** pattern with **property-based testing** in Java. The Object Mother guaranteed structural validity so jqwik's generated data never wasted runs on impossible objects, while jqwik explored the input space far more thoroughly than any hand-written example could.

The payoff is tests that describe *what must always be true* rather than enumerating inputs — and a workflow where failures shrink to minimal, reproducible counterexamples instead of intimidating random blobs.

As always, the full source code is available over on GitHub.
