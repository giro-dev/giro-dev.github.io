---
title: "Object Mother meets property-based testing in Java"
date: 2026-07-21T00:00:00Z
description: "Combining the Object Mother pattern with property-based testing to generate valid yet varied Java test data."
---

## Problem

The Object Mother pattern (as implemented in [Matriarch](/blog/matriarch-object-mother-java/)) makes it easy to build valid domain objects with sensible defaults. But example-based tests still exercise a handful of hand-picked values, so edge cases hide in the gaps between the examples we happened to think of.

Property-based testing flips this around: instead of asserting on fixed inputs, you assert that a property holds for *many* generated inputs. The tension is that random generators tend to produce structurally invalid objects, which drowns the interesting cases in noise.

## Approach

Use the Object Mother to guarantee structural validity and use a property-based engine (jqwik) to vary the parts that matter. The Mother owns the invariants; jqwik owns the exploration.

```java
@Property
void discountNeverExceedsOrderTotal(@ForAll("orders") Order order) {
    Money discount = pricing.discountFor(order);
    assertThat(discount).isLessThanOrEqualTo(order.total());
}

@Provide
Arbitrary<Order> orders() {
    Arbitrary<Integer> lineCount = Arbitraries.integers().between(1, 20);
    return lineCount.map(n -> OrderMother.anOrder()
            .withLines(n)
            .build());
}
```

The Mother keeps every generated `Order` internally consistent (line totals sum to the order total, currencies match), while jqwik sweeps the dimension that actually stresses the property — the number of lines.

## Key decisions

- **Mother owns invariants, generator owns variation** — the arbitrary only tweaks fields that are safe to randomise, so shrinking stays meaningful.
- **Deterministic seeds** — jqwik reports the failing seed, and Matriarch's seeded generation reproduces the exact object, so failures are replayable.
- **Shrinking-friendly builders** — expose narrow `Arbitrary`s (counts, enums, ranges) rather than fully random graphs, so jqwik can shrink to a minimal counterexample.
- **Properties over examples for policies** — pricing, validation and serialization round-trips are expressed as invariants; example tests stay for documentation of specific scenarios.

## Outcome

Tests describe *what must always be true* instead of enumerating inputs, and the combination surfaces edge cases (empty lines, boundary quantities, currency mismatches) that fixed fixtures missed — without giving up the readability and validity guarantees of the Object Mother.
