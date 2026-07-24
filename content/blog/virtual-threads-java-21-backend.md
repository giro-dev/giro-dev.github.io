---
title: "Virtual threads in Java 21: what changes for backend platforms"
date: 2026-07-24T00:00:00Z
description: "When Java 21 virtual threads help high-throughput Spring Boot services, and when they don't."
---

## Problem

Backend platforms have leaned on bounded thread pools for a decade. Each request that performs blocking I/O — a database call, a downstream HTTP request — parks a precious platform thread. Under load, pools saturate, latency climbs, and the usual fix is more pools, bigger pools and careful tuning. Java 21 makes virtual threads a stable feature, and the promise is tempting: cheap threads that let you write straightforward blocking code without paying for the blocking. The risk is treating them as a drop-in performance switch and getting surprised in production.

## Approach

A virtual thread is a lightweight thread scheduled by the JVM onto a small pool of platform ("carrier") threads. When a virtual thread blocks on I/O, the JVM unmounts it from its carrier and frees that carrier for other work. Throughput scales with the number of *concurrent operations*, not the number of OS threads.

In Spring Boot 3.2+, enabling them is mostly configuration:

```properties
spring.threads.virtual.enabled=true
```

This switches the servlet request executor (Tomcat) to per-request virtual threads, so each incoming request runs on its own virtual thread. The mental model becomes "one virtual thread per task, block freely" instead of "share a scarce pool, never block".

## Key decisions

- **Use them for I/O-bound request handling, not CPU-bound work.** Virtual threads win when threads spend most of their time waiting. For CPU-bound work you still need a bounded pool sized to your cores — spawning a million virtual threads to crunch numbers just adds scheduling overhead.
- **Watch for pinning.** A virtual thread that blocks inside a `synchronized` block or a native call stays *pinned* to its carrier and cannot unmount, silently reintroducing the pool-exhaustion problem. Prefer `ReentrantLock` over `synchronized` on hot paths, and use `-Djdk.tracePinnedThreads=full` to find offenders. (JDK 24 relaxes most `synchronized` pinning, but on 21 LTS it still bites.)
- **Don't pool virtual threads.** They are cheap to create and meant to be short-lived and per-task. Pooling them defeats the point and reintroduces the very contention you were escaping. Use `Executors.newVirtualThreadPerTaskExecutor()`.
- **Bound your dependencies, not your threads.** With unbounded concurrency, a connection pool (HikariCP), a downstream API rate limit or a `Semaphore` becomes your real backpressure mechanism. Size those deliberately, because the thread count no longer protects them.
- **ThreadLocal caution.** Millions of virtual threads each holding heavy `ThreadLocal` state can blow up memory. For request-scoped context, prefer scoped values or explicit propagation.

## Outcome

Virtual threads are best understood as a *concurrency scalability* feature, not a raw speed-up. For blocking, I/O-heavy Spring Boot services they remove thread-pool tuning as a bottleneck and keep the simple synchronous programming model. They do nothing for CPU-bound workloads, and they shift the burden of backpressure onto your connection pools and downstream limits. Adopt them where requests mostly wait, audit for pinning before rolling out, and treat every external dependency as the new capacity limit.
