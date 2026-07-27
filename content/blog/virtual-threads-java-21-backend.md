---
title: "Virtual threads in Java 21: a practical guide for backend platforms"
date: 2026-07-24T00:00:00Z
description: "A hands-on tutorial on Java 21 virtual threads: how they work, how to enable them in Spring Boot, and the pitfalls to avoid."
---

## Problem

Backend platforms have leaned on bounded thread pools for a decade. Each request that performs blocking I/O — a database call, a downstream HTTP request — parks a precious platform thread. Under load, pools saturate, latency climbs, and the usual fix is more pools, bigger pools and careful tuning. Java 21 makes virtual threads a stable feature, and the promise is tempting: cheap threads that let you write straightforward blocking code without paying for the blocking. The risk is treating them as a drop-in performance switch and getting surprised in production.

This post is a practical, step-by-step guide: what virtual threads are, how to enable them, how to verify they work, and the traps to avoid. Treat it as a reference you can come back to.

## Background: how virtual threads work

A virtual thread is a lightweight thread scheduled by the JVM onto a small pool of platform ("carrier") threads. When a virtual thread blocks on I/O, the JVM *unmounts* it from its carrier and frees that carrier for other work. When the I/O completes, the virtual thread is *remounted* on any available carrier and resumes.

```text
Platform threads (classic)          Virtual threads (Java 21)
------------------------            -------------------------
1 request  -> 1 OS thread           1 request  -> 1 virtual thread
blocked I/O holds the OS thread     blocked I/O unmounts the vthread
throughput bound by pool size       throughput bound by concurrent ops
```

The key shift in mental model: with platform threads you *share a scarce pool and avoid blocking*; with virtual threads you spawn *one thread per task and block freely*. Throughput scales with the number of concurrent operations, not the number of OS threads.

## Step 1 — Create a virtual thread

The low-level API lives on `Thread`:

```java
// Start a single virtual thread
Thread.startVirtualThread(() -> System.out.println("hello from " + Thread.currentThread()));

// Or with the builder, for naming and lifecycle control
Thread t = Thread.ofVirtual().name("worker-1").start(() -> doWork());
t.join();
```

For running many tasks, use the dedicated executor. It creates a **new virtual thread per task** — do not confuse it with a fixed pool:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = ids.stream()
        .map(id -> executor.submit(() -> fetchUser(id)))  // each runs on its own vthread
        .toList();
    for (var f : futures) {
        System.out.println(f.get());
    }
} // try-with-resources waits for all tasks to finish
```

## Step 2 — Enable them in Spring Boot

In Spring Boot 3.2+, per-request virtual threads are a single property:

```properties
spring.threads.virtual.enabled=true
```

This switches the servlet request executor (Tomcat) so each incoming HTTP request runs on its own virtual thread. If you need it programmatically, or for `@Async` and other executors:

```java
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

## Step 3 — Verify it actually works

Print the current thread on a request handler and load-test it. A virtual thread prints as `VirtualThread[#NN]/runnable@ForkJoinPool-1-worker-M`:

```java
@GetMapping("/whoami")
String whoami() {
    return Thread.currentThread().toString();
    // => VirtualThread[#42]/runnable@ForkJoinPool-1-worker-3
}
```

Fire many concurrent slow requests (e.g. a handler that sleeps 500ms) and confirm throughput scales far beyond the old `server.tomcat.threads.max`. With platform threads you would plateau at the pool size; with virtual threads you should serve thousands of concurrent slow requests on a handful of carriers.

## Common pitfalls (and how to detect them)

- **Pinning.** A virtual thread that blocks inside a `synchronized` block or a native call stays *pinned* to its carrier and cannot unmount — silently reintroducing pool exhaustion. Prefer `ReentrantLock` over `synchronized` on hot paths. Detect it with:

  ```bash
  java -Djdk.tracePinnedThreads=full -jar app.jar
  ```

  (JDK 24 relaxes most `synchronized` pinning, but on 21 LTS it still bites.)

- **Don't pool virtual threads.** They are cheap to create and meant to be short-lived and per-task. Wrapping them in a fixed-size pool reintroduces the contention you were escaping. Always use `newVirtualThreadPerTaskExecutor()`.

- **CPU-bound work.** Virtual threads win only when threads spend most of their time *waiting*. For CPU-bound work you still want a bounded pool sized to your cores — spawning a million virtual threads to crunch numbers just adds scheduling overhead.

- **Backpressure moves.** With unbounded concurrency, your connection pool (HikariCP), a downstream rate limit or an explicit `Semaphore` becomes the real capacity limit. Size those deliberately — the thread count no longer protects them.

  ```java
  Semaphore limit = new Semaphore(100); // cap concurrent downstream calls
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
      for (var task : tasks) {
          executor.submit(() -> {
              limit.acquire();
              try { callDownstream(task); } finally { limit.release(); }
          });
      }
  }
  ```

- **ThreadLocal memory.** Millions of virtual threads each holding heavy `ThreadLocal` state can blow up memory. For request-scoped context, prefer scoped values or explicit propagation.

## Outcome

Virtual threads are best understood as a *concurrency scalability* feature, not a raw speed-up. For blocking, I/O-heavy Spring Boot services they remove thread-pool tuning as a bottleneck and keep the simple synchronous programming model. They do nothing for CPU-bound workloads, and they shift the burden of backpressure onto your connection pools and downstream limits.

A checklist before rolling out:

1. Confirm the workload is I/O-bound.
2. Enable `spring.threads.virtual.enabled=true` (or wire the executor).
3. Verify handlers run on `VirtualThread[...]`.
4. Audit hot paths for `synchronized`/native pinning with `-Djdk.tracePinnedThreads=full`.
5. Size connection pools and add explicit backpressure where needed.

Adopt them where requests mostly wait, audit for pinning first, and treat every external dependency as the new capacity limit.
