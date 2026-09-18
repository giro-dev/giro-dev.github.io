---
title: "Structured concurrency and scoped values in Java 25: replacing ThreadLocal in request pipelines"
date: 2026-08-22T00:00:00Z
description: "A hands-on tutorial on the Java 25 StructuredTaskScope and the ScopedValue API: fanning out calls safely and propagating request context without ThreadLocal."
aliases:
  - /blog/structured-concurrency-scoped-values-java-21/
---
# Structured concurrency and scoped values in Java 25: replacing ThreadLocal in request pipelines

## 1. Overview

Virtual threads make it cheap to run many tasks. Two older patterns still cause problems in request pipelines: starting fan-out tasks without a shared scope, and storing request context in `ThreadLocal`.

In this post I will show how `StructuredTaskScope` and `ScopedValue` solve these problems in Java 25. I will also show how to move from the Java 21 preview API and which problems still need attention.

Java 25 (LTS) is the baseline for this post:

| Feature | Status in Java 21 | Status in Java 25 |
|---|---|---|
| Scoped values | Preview (JEP 446) | **Final** (JEP 506) — no flag needed |
| Structured concurrency | Preview (JEP 453) | Preview, redesigned API (JEP 505) |
| `synchronized` pinning virtual threads | Yes | **No** — fixed in Java 24 (JEP 491) |

## 2. Background: what structured concurrency gives me

Structured concurrency uses the same idea as structured programming. Work starts inside a block and ends inside that block:

```text
Unstructured (executor)                Structured (StructuredTaskScope)
-----------------------                --------------------------------
submit() escapes the method            forks live inside the try block
failure of one task is invisible       first failure cancels siblings
cancellation is manual bookkeeping     scope close() joins everything
stack traces lose the caller           parent/child relation is explicit
```

The rule is simple: **when the block exits, no forked task is still running.** There are no orphan tasks or leaked tasks.

## 3. Step 1: fan out with `StructuredTaskScope`

In Java 25, I open a scope with the static factory `StructuredTaskScope.open()`. The public constructors and the `ShutdownOnFailure` and `ShutdownOnSuccess` subclasses from the Java 21 preview are gone. The no-argument factory covers the common case: wait for all subtasks to succeed and cancel everything on the first failure.

```java
record Dashboard(User user, List<Order> orders) {}

Dashboard loadDashboard(long userId) throws InterruptedException {
    try (var scope = StructuredTaskScope.open()) {
        Subtask<User> user = scope.fork(() -> userClient.fetch(userId));
        Subtask<List<Order>> orders = scope.fork(() -> orderClient.fetchFor(userId));

        scope.join();   // throws FailedException if either subtask failed

        return new Dashboard(user.get(), orders.get());
    }
}
```

Compared with the Java 21 preview, three things changed and one part became simpler:

- `new StructuredTaskScope.ShutdownOnFailure()` changed to `StructuredTaskScope.open()`.
- `scope.join(); scope.throwIfFailed();` changed to one `scope.join()`, which throws `StructuredTaskScope.FailedException` with the subtask exception as its cause.
- `scope.joinUntil(instant)` changed to a timeout in the scope configuration in Step 5.

I still get these behaviours:

1. If `userClient.fetch` throws, the `orders` subtask is **interrupted immediately**. This avoids a wasted downstream call.
2. `join()` returns only when every subtask is done. `close()` also waits for all threads, so nothing outlives the method.
3. I call `Subtask::get` only **after** a successful `join()`. Before that it throws `IllegalStateException`.

Each fork runs on its own virtual thread. A fan-out of 50 is therefore as cheap as a fan-out of 2.

I handle failures by catching `FailedException` and matching its cause:

```java
try (var scope = StructuredTaskScope.open()) {
    // ...
} catch (StructuredTaskScope.FailedException e) {
    switch (e.getCause()) {
        case IOException ioe -> throw new DownstreamUnavailable(ioe);
        default -> throw e;
    }
}
```

## 4. Step 2: choose a `Joiner` instead of a shutdown policy

The policy is no longer in a subclass. I pass a `Joiner` to `open`. Four factories cover most cases:

```java
// "AND": all results required, stream of completed subtasks, fail fast
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allSuccessfulOrThrow())) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> quotes = scope.join().map(Subtask::get).toList();
    return cheapest(quotes);
}

// "AND", heterogeneous subtasks: no scope result, read via Subtask::get
try (var scope = StructuredTaskScope.open(Joiner.awaitAllSuccessfulOrThrow())) { ... }

// "OR" / race: first success wins, losers are cancelled
try (var scope = StructuredTaskScope.open(Joiner.<Quote>anySuccessfulResultOrThrow())) {
    scope.fork(() -> providerA.quote(req));
    scope.fork(() -> providerB.quote(req));
    return scope.join();          // fastest successful quote
}

// "best effort": wait for everything, never throw, inspect each outcome
try (var scope = StructuredTaskScope.open(Joiner.<Quote>awaitAll())) { ... }
```

`Joiner.allUntil(Predicate)` is the base for custom policies. It yields all subtasks and cancels the scope when the predicate returns `true`. If the predicate never cancels, I can filter the result to keep successful tasks and ignore the rest:

```java
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allUntil(sub -> false))) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> successes = scope.join()
        .filter(sub -> sub.state() == Subtask.State.SUCCESS)
        .map(Subtask::get)
        .toList();
}
```

For another policy, I implement `Joiner` directly. `onFork` and `onComplete` return a `boolean` that can cancel the scope. `result()` creates the value returned by `join()`. Both callbacks run on subtask threads, so the implementation must be thread safe. I also create a new `Joiner` for every scope.

## 5. Step 3: configure timeouts, names, and thread factories

The two-argument `open` takes a function over the default `Configuration`. This is where the Java 21 `joinUntil` deadline moved. The timeout now covers the whole fan-out, including cancellation:

```java
try (var scope = StructuredTaskScope.open(
        Joiner.awaitAllSuccessfulOrThrow(),
        cf -> cf.withName("dashboard")                       // shows up in thread dumps
                .withTimeout(Duration.ofMillis(300))         // cancels the scope on expiry
                .withThreadFactory(Thread.ofVirtual().name("dash-", 0).factory()))) {

    var a = scope.fork(() -> serviceA.call());
    var b = scope.fork(() -> serviceB.call());

    scope.join();            // TimeoutException (as the cause) if the budget is blown
    return merge(a.get(), b.get());
}
```

Naming the scope and its threads helps when I read a structured thread dump:
`jcmd <pid> Thread.dump_to_file -format=json dump.json`. The dump groups subtasks under their owner. A name makes a production stall easier to read.

## 6. Step 4: replace `ThreadLocal` with `ScopedValue`

`ScopedValue` is final in Java 25. It needs no `--enable-preview` flag and is safe to expose in library signatures. A value is bound for one call, cannot change inside that call, and is unbound automatically when the call returns:

```java
public final class RequestContext {
    public static final ScopedValue<TenantId> TENANT = ScopedValue.newInstance();
    private RequestContext() {}
}

// At the edge (filter / interceptor): bind for the request
ScopedValue.where(RequestContext.TENANT, tenantId)
           .run(() -> chain.doFilter(request, response));

// Deep in the call stack: read it
TenantId tenant = RequestContext.TENANT.get();
```

I can bind several values by chaining on the carrier. I use `call` when the operation returns a value:

```java
Report report = ScopedValue.where(RequestContext.TENANT, tenantId)
                           .where(RequestContext.CORRELATION_ID, correlationId)
                           .call(() -> reportService.build());
```

The API shape is important. The static `ScopedValue.runWhere` and `callWhere` helpers from earlier previews were removed. In Java 25, I use `where(...).run(...)` and `where(...).call(...)`.

The important differences from `ThreadLocal` are:

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Lifetime | until removed (or thread dies) | the dynamic scope of `run`/`call` |
| Mutability | `set()` anywhere | immutable inside the scope |
| Cleanup | manual `remove()`, leaks if forgotten | automatic on scope exit |
| Inheritance | only via `InheritableThreadLocal`, copies the value | inherited by `StructuredTaskScope` forks, no copy |
| Cost per thread | a map entry per thread | shared, read-only binding |

When a binding is not guaranteed, I read it defensively:

```java
TenantId tenant = RequestContext.TENANT.orElse(TenantId.SYSTEM);   // Java 25: argument must not be null
if (RequestContext.TENANT.isBound()) { /* ... */ }
```

A nested call can bind a different value. It shadows the outer value only in that call and does not change the outer binding:

```java
ScopedValue.where(RequestContext.TENANT, otherTenant)
           .run(() -> migrationJob.copyFrom());   // outer binding intact afterwards
```

## 7. Step 5: combine both features

This is where the two features work together. Every subtask forked inside the scope inherits scoped values. There is no copying and no argument plumbing:

```java
Report buildReport(TenantId tenantId) throws InterruptedException {
    return ScopedValue.where(RequestContext.TENANT, tenantId).call(() -> {
        try (var scope = StructuredTaskScope.open()) {
            var sales = scope.fork(this::loadSales);       // sees TENANT
            var costs = scope.fork(this::loadCosts);       // sees TENANT

            scope.join();
            return new Report(sales.get(), costs.get());
        }
    });
}

private Sales loadSales() {
    TenantId tenant = RequestContext.TENANT.get();   // inherited, no argument passing
    return salesRepository.findFor(tenant);
}
```

With `ThreadLocal`, this required `InheritableThreadLocal` and a task-decorating executor. It also broke silently when work was sent to another pool.

## 8. Step 6: migrate an existing pipeline

I use this order for an incremental migration:

1. **Inventory** the `ThreadLocal`s and their `set` calls:

   ```bash
   grep -rn "ThreadLocal\|InheritableThreadLocal" src/main/java
   ```

2. **Classify** each value. A request-scoped value that is read-only after the edge can use `ScopedValue`. A mutable accumulator needs an explicit mutable object because scoped values cannot be reassigned.
3. **Bind at the edges only.** Use a servlet filter, message listener, or scheduled job entry point. Use one binding site per entry point, never business code.
4. **Convert fan-outs.** Replace groups of `executor.submit(...)` and `Future.get()` with a `StructuredTaskScope`. Choose the `Joiner` from Step 4.
5. **Delete the decorators.** Remove `TaskDecorator`s, MDC-copying wrappers, and `InheritableThreadLocal` code that existed only to move context between threads.
6. **Keep MDC bridged** when using SLF4J. Logging frameworks still read the MDC, so set it from the scoped value at the edge:

   ```java
   ScopedValue.where(RequestContext.TENANT, tenantId).run(() -> {
       MDC.put("tenant", tenantId.value());
       try { chain.doFilter(request, response); } finally { MDC.clear(); }
   });
   ```

7. **Enable preview for structured concurrency only.** `ScopedValue` does not need it:

   ```xml
   <plugin>
     <artifactId>maven-compiler-plugin</artifactId>
     <configuration>
       <release>25</release>
       <compilerArgs><arg>--enable-preview</arg></compilerArgs>
     </configuration>
   </plugin>
   ```

   ```bash
   java --enable-preview -jar app.jar
   ```

   Preview classes have a minor-version marker. Tests and runtime must therefore use the same JDK feature release. If I cannot ship `--enable-preview`, I can split the migration: adopt `ScopedValue` now and keep executors for fan-out.

### Migration cheat sheet: Java 21 preview → Java 25

```diff
- try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
+ try (var scope = StructuredTaskScope.open()) {
      var a = scope.fork(() -> callA());
      var b = scope.fork(() -> callB());
-     scope.join();
-     scope.throwIfFailed();
+     scope.join();                        // throws FailedException
      return merge(a.get(), b.get());
  }

- new StructuredTaskScope.ShutdownOnSuccess<Quote>()
+ StructuredTaskScope.open(Joiner.<Quote>anySuccessfulResultOrThrow())

- scope.joinUntil(Instant.now().plusMillis(300));
+ StructuredTaskScope.open(joiner, cf -> cf.withTimeout(Duration.ofMillis(300)));

- override handleComplete(Subtask) in a scope subclass
+ implement Joiner.onComplete(Subtask) / result(), or use Joiner.allUntil(predicate)
```

## 9. Common pitfalls and how to detect them

- **Do not leak a `Subtask` outside the scope.** Calling `get()` after the `try` block or storing a subtask in a field defeats the model. Read results inside the block and return plain values.
- **Make subtasks respond to interrupts.** Cancellation is cooperative. A subtask in a non-interruptible call can delay `close()` indefinitely because `close()` always waits for its threads. Use interruptible downstream clients and give them their own timeouts.
- **Use a scope from its owner thread only.** The owner thread must call `fork`, `join`, and `close`. `join` can run only once, and `fork` cannot run after `join`. Do not store a scope in a bean field; create it for each call.
- **Create a new `Joiner` for every scope.** A `Joiner` is stateful, so never share or cache it.
- **Do not expect `ScopedValue` to be mutable.** There is no `set()`. Pass a mutable holder explicitly or restructure the code to return values.
- **Handle unbound reads in background jobs.** `get()` on an unbound scoped value throws `NoSuchElementException`. Scheduled jobs and Kafka listeners are separate entry points and need their own binding. Test this path:

  ```java
  assertThatThrownBy(() -> RequestContext.TENANT.get())
      .isInstanceOf(NoSuchElementException.class);
  ```

  `orElse(null)` is also rejected in Java 25. Use `isBound()` when absence is valid.
- **Do not assume pinning is still the main problem.** Since Java 24 (JEP 491), blocking in `synchronized` no longer pins a virtual thread. Native frames and class-initialization blocking can still pin. Watch the `jdk.VirtualThreadPinned` JFR event instead of the removed system property.
- **Backpressure is still needed.** Structured fan-out makes it easy to create concurrency. Connection pools and downstream rate limits remain the capacity limit. Use a `Semaphore` or a bounded pool when needed.
- **Expect preview API changes.** Structured concurrency is on its fifth preview in Java 25, and a sixth is planned for Java 26. Put fan-out behind a small internal helper such as `Parallel.all(...)`, so an upgrade changes one class instead of many call sites.

## 10. Conclusion

The two features solve different parts of the same problem. `StructuredTaskScope` bounds concurrency by scope: failures cancel siblings, one timeout covers the fan-out, and no task outlives the method. `ScopedValue`, which is final in Java 25, makes context immutable and bounded. Request metadata reaches forked subtasks without decorators and without per-thread state for every virtual thread.

Before I adopt them, I use this checklist:

1. Move to Java 25 (LTS). `ScopedValue` needs no flag and `synchronized` no longer pins carriers.
2. Move request-scoped, read-only `ThreadLocal`s to `ScopedValue`, bound only at entry points.
3. Replace `executor.submit` and `Future.get()` groups with `StructuredTaskScope.open(...)` and an explicit `Joiner`.
4. Move per-call timeouts to a scope-wide `withTimeout`, and name scopes so thread dumps stay readable.
5. Bridge the logging MDC at the edge and remove inheritance hacks and task decorators.
6. Test the unbound path for jobs and listeners, and test cancellation.
7. Keep the preview structured-concurrency API behind one helper class so upgrades stay small.

Virtual threads make concurrency cheap. Structured concurrency and scoped values make it safer. I use them where a request fans out, and I bind context explicitly at every entry point.
