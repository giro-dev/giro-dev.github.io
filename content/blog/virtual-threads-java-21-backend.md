---
title: "Virtual threads in Java 21: a practical guide for backend platforms"
date: 2026-07-24T00:00:00Z
description: "A hands-on tutorial on Java 21 virtual threads: how they work, how to enable them in Spring Boot, and the pitfalls to avoid."
---
# Virtual threads in Java 21: a practical guide for backend platforms

## 1. Overview

Backend applications often use bounded thread pools. A request that waits for a database or another HTTP service keeps a platform thread busy. When the pool is full, latency grows.

In this post I will show how virtual threads work, how to enable them, how to check them, and which problems to avoid. Java 21 makes virtual threads a stable feature, but they are not a general performance switch.

## 2. Background: how virtual threads work

A virtual thread is a light thread managed by the JVM. The JVM runs it on a small pool of platform threads called carrier threads. When a virtual thread waits for I/O, the JVM removes it from its carrier. Another task can then use that carrier. When the I/O is ready, the virtual thread runs again on an available carrier.

```text
Platform threads (classic)          Virtual threads (Java 21)
------------------------            -------------------------
1 request  -> 1 OS thread           1 request  -> 1 virtual thread
blocked I/O holds the OS thread     blocked I/O unmounts the vthread
throughput bound by pool size       throughput bound by concurrent ops
```

The main change is how I think about threads:

- With platform threads, I share a small pool and try to avoid blocking.
- With virtual threads, I create one thread per task and can block during I/O.

Throughput is then limited by the number of concurrent operations, not only by the number of OS threads.

## 3. Step 1: create a virtual thread

The low-level API is on `Thread`:

```java
// Start a single virtual thread
Thread.startVirtualThread(() -> System.out.println("hello from " + Thread.currentThread()));

// Or with the builder, for naming and lifecycle control
Thread t = Thread.ofVirtual().name("worker-1").start(() -> doWork());
t.join();
```

For many tasks, use the dedicated executor. It creates a **new virtual thread per task**. It is not a fixed pool:

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

## 4. Step 2: enable them in Spring Boot

In Spring Boot 3.2 and later, enable virtual threads with one property:

```properties
spring.threads.virtual.enabled=true
```

This changes the Tomcat servlet request executor. Each incoming HTTP request runs on its own virtual thread. For `@Async` and other executors, I can configure one directly:

```java
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

## 5. Step 3: verify that it works

I can print the current thread from a request handler and then run a load test. A virtual thread looks like `VirtualThread[#NN]/runnable@ForkJoinPool-1-worker-M`:

```java
@GetMapping("/whoami")
String whoami() {
    return Thread.currentThread().toString();
    // => VirtualThread[#42]/runnable@ForkJoinPool-1-worker-3
}
```

I can also send many slow requests, for example requests that sleep for 500ms. With platform threads, throughput stops at the pool size. With virtual threads, the service should handle thousands of slow concurrent requests on a small number of carriers. This should be much higher than the old `server.tomcat.threads.max` limit.

## 6. Common pitfalls and how to detect them

- **Pinning.** A virtual thread that blocks inside a `synchronized` block or a native call stays on its carrier. It cannot unmount, so the carrier pool can run out of capacity. On hot paths, prefer `ReentrantLock` over `synchronized`. On Java 21 LTS, use this command to find pinning:

  ```bash
  java -Djdk.tracePinnedThreads=full -jar app.jar
  ```

  JDK 24 relaxes most `synchronized` pinning, but it still affects Java 21.

- **Do not pool virtual threads.** Virtual threads are cheap and should be short-lived and created per task. A fixed-size pool brings back the contention that virtual threads were meant to remove. Use `newVirtualThreadPerTaskExecutor()`.

- **CPU-bound work.** Virtual threads help when most of the time is spent waiting. CPU-bound work still needs a bounded pool sized for the available cores. A million virtual threads doing calculations only add scheduling overhead.

- **Backpressure moves.** With unbounded concurrency, the connection pool, a downstream rate limit, or an explicit `Semaphore` becomes the real limit. I must size those limits deliberately because the thread count no longer protects them.

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

- **ThreadLocal memory.** Millions of virtual threads with heavy `ThreadLocal` state can use too much memory. For request context, prefer scoped values or explicit propagation.

## 7. Conclusion

Virtual threads improve concurrency scalability. They are not a direct speed-up for every workload. They help Spring Boot services that spend much of their time waiting for I/O, because the service no longer depends on large platform thread pools.

They do not improve CPU-bound work. They also move the backpressure problem to connection pools and downstream limits.

Before I enable them, I use this checklist:

1. Confirm that the workload is I/O-bound.
2. Enable `spring.threads.virtual.enabled=true` or configure the executor.
3. Check that handlers run on `VirtualThread[...]`.
4. Check hot paths for `synchronized` or native pinning with `-Djdk.tracePinnedThreads=full`.
5. Size connection pools and add explicit backpressure where needed.

I use virtual threads when requests mostly wait. I check pinning first and treat every external dependency as a capacity limit.
