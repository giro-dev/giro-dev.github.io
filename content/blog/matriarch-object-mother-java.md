---
title: "Matriarch: test data generation for Java"
date: 2026-07-21T00:00:00Z
description: "Building a fluent Object Mother library to speed up Java tests."
---

## Problem

Writing test fixtures for Java POJOs and Records is repetitive and error-prone. As models grow, constructors and builders explode, and tests become cluttered with boilerplate.

## Approach

I implemented the **Object Mother** pattern as a small open-source Java library with a fluent API. Users define a `Mother` for a class once, then create valid instances with selective overrides.

## Key decisions

- **Fluent API** — chaining keeps test setup readable.
- **Jackson + SnakeYAML** — pattern loading from YAML keeps fixtures external and reusable.
- **Deterministic seeds** — reproducible data generation for stable tests.
- **JUnit 5 integration** — `@MotherFactoryResource` and `@RandomArg` for data-driven tests.
- **Maven Central + GitHub Actions** — automated publishing and CI.

## Outcome

A reusable library that reduces fixture boilerplate and keeps Java tests focused on behaviour rather than object construction.
