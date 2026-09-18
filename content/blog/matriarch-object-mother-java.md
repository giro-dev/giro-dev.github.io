---
title: "Matriarch: test data generation for Java"
date: 2026-07-21T00:00:00Z
description: "Building a fluent Object Mother library to speed up Java tests."
---
# Matriarch: test data generation for Java

## 1. Overview

Java tests often need many objects before they can test the real behaviour. Creating those objects by hand takes time and makes the tests harder to read.

In this post I will show the idea behind **Matriarch**, a small Java library that creates test data with the Object Mother pattern.

## 2. Problem

Writing test fixtures for Java POJOs and Records is repetitive and can introduce mistakes. As models grow, constructors and builders also grow, and tests contain more setup code.

## 3. Approach

I implemented the **Object Mother** pattern as a small open-source Java library with a fluent API. Users define a `Mother` for a class once. They can then create valid instances and override only the values needed by a test.

## 4. Key decisions

- **Fluent API** — method chaining keeps test setup readable.
- **Jackson + SnakeYAML** — YAML patterns keep fixtures external and reusable.
- **Deterministic seeds** — the same seed produces the same data, so tests stay stable.
- **JUnit 5 integration** — `@MotherFactoryResource` and `@RandomArg` support data-driven tests.
- **Maven Central + GitHub Actions** — publishing and CI run automatically.

## 5. Outcome

Matriarch is a reusable library that reduces fixture setup. Tests can focus on behaviour instead of object construction.

## 6. Conclusion

The Object Mother pattern gives Java tests one place for common test data. Matriarch adds a fluent API, external patterns, and deterministic data so the setup stays small and repeatable.
