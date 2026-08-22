---
title: "Structured concurrency and scoped values in Java 25: replacing ThreadLocal in request pipelines"
date: 2026-08-22T00:00:00Z
description: "A hands-on tutorial on the Java 25 StructuredTaskScope and the finalized ScopedValue API: fanning out calls safely and propagating request context without ThreadLocal."
aliases:
  - /blog/structured-concurrency-scoped-values-java-21/
---

## Problem

Once a service runs on virtual threads, two old habits start to hurt.

The first is **unstructured fan-out**. A request handler submits three downstream calls to an executor, collects the futures and waits. If one call fails, the others keep running: wasted work, leaked connections, and a latency floor set by the slowest sibling. Cancellation is manual, and error handling is a pile of `try/catch` around `Future.get()`.

The second is **`ThreadLocal` context propagation**. Tenant id, correlation id and user principal are traditionally stashed in a `ThreadLocal`. That works when threads are scarce and pooled — and breaks in two ways when they are not: the value is invisible to the child tasks you fan out to, and with millions of short-lived virtual threads each holding mutable per-thread state, memory and lifecycle become a real problem.

Java 25 (LTS) is the right baseline to fix both:

| Feature | Status in Java 21 | Status in Java 25 |
|---|---|---|
| Scoped values | Preview (JEP 446) | **Final** (JEP 506) — no flag needed |
| Structured concurrency | Preview (JEP 453) | Preview, redesigned API (JEP 505) |
| `synchronized` pinning virtual threads | Yes | **No** — fixed in Java 24 (JEP 491) |

This post is a step-by-step guide to both APIs *as they look in Java 25*, with the migration path from the Java 21 shape and the traps that remain.

## Background: what "structured" buys you

Structured concurrency applies the rule that made structured programming work — *control flow enters and leaves a block at one place* — to threads:

```text
Unstructured (executor)                Structured (StructuredTaskScope)
-----------------------                --------------------------------
submit() escapes the method            forks live inside the try block
failure of one task is invisible       first failure cancels siblings
cancellation is manual bookkeeping     scope close() joins everything
stack traces lose the caller           parent/child relation is explicit
```

The invariant: **when the block exits, no forked task is still running.** No orphans, no leaks.

## Step 1 — Fan out with `StructuredTaskScope`

In Java 25 a scope is opened with the static factory `StructuredTaskScope.open()` — the public constructors and the `ShutdownOnFailure` / `ShutdownOnSuccess` subclasses from the Java 21 preview are gone. The zero-arg factory covers the common case: *wait for all subtasks to succeed, cancel everything on the first failure*.

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

Compared with the Java 21 preview, three things changed and one thing got simpler:

- `new StructuredTaskScope.ShutdownOnFailure()` → `StructuredTaskScope.open()`.
- `scope.join(); scope.throwIfFailed();` → a single `scope.join()`, which throws `StructuredTaskScope.FailedException` with the subtask's exception as the cause.
- `scope.joinUntil(instant)` → a timeout is now part of the scope's *configuration* (Step 3).

What you still get for free:

1. If `userClient.fetch` throws, the `orders` subtask is **interrupted immediately** — no wasted downstream call.
2. `join()` returns only when every subtask is done, and `close()` waits for the threads regardless, so nothing outlives the method.
3. Only call `Subtask::get` **after** a successful `join()`; before that it throws `IllegalStateException`.

Each `fork` runs on its own virtual thread, so a fan-out of 50 is as cheap as a fan-out of 2.

Handle failures by catching `FailedException` and pattern-matching the cause:

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

## Step 2 — Choose a `Joiner` instead of a shutdown policy

Policy no longer lives in a subclass; it lives in a `Joiner` passed to `open`. Four factories cover almost everything:

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

`Joiner.allUntil(Predicate)` is the building block for custom policies: it yields *all* subtasks and cancels the scope as soon as your predicate returns `true`. With a predicate that never cancels, the classic "return what succeeded, ignore the rest" is a filter over the resulting stream:

```java
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allUntil(sub -> false))) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> successes = scope.join()
        .filter(sub -> sub.state() == Subtask.State.SUCCESS)
        .map(Subtask::get)
        .toList();
}
```

For anything more exotic, implement `Joiner` directly: `onFork` and `onComplete` return a `boolean` that cancels the scope, and `result()` produces what `join()` returns. Both callbacks run on subtask threads, so implementations must be thread safe — and a `Joiner` instance must never be reused across scopes.

## Step 3 — Configure timeouts, names and thread factories

The 2-arg `open` takes a function over the default `Configuration`. This is where the Java 21 `joinUntil` deadline went, and it now covers the *whole* fan-out including cancellation:

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

Naming the scope and its threads is not cosmetic: structured thread dumps (`jcmd <pid> Thread.dump_to_file -format=json dump.json`) group subtasks under their owner, so a named scope is what makes a production stall readable.

## Step 4 — Replace `ThreadLocal` with `ScopedValue` (now final)

`ScopedValue` is a permanent API in Java 25: no `--enable-preview`, safe to expose in library signatures. A value is bound for the duration of a call, immutable inside it, and unbound automatically when the call returns:

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

Bind several values by chaining on the carrier, and use `call` when the operation returns a value:

```java
Report report = ScopedValue.where(RequestContext.TENANT, tenantId)
                           .where(RequestContext.CORRELATION_ID, correlationId)
                           .call(() -> reportService.build());
```

Note the API shape: the static `ScopedValue.runWhere`/`callWhere` helpers that existed in earlier previews were removed — `where(...).run(...)` / `where(...).call(...)` is the only form in Java 25.

The differences that matter versus `ThreadLocal`:

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Lifetime | until removed (or thread dies) | the dynamic scope of `run`/`call` |
| Mutability | `set()` anywhere | immutable inside the scope |
| Cleanup | manual `remove()`, leaks if forgotten | automatic on scope exit |
| Inheritance | only via `InheritableThreadLocal`, copies the value | inherited by `StructuredTaskScope` forks, no copy |
| Cost per thread | a map entry per thread | shared, read-only binding |

Read defensively where a binding is not guaranteed:

```java
TenantId tenant = RequestContext.TENANT.orElse(TenantId.SYSTEM);   // Java 25: argument must not be null
if (RequestContext.TENANT.isBound()) { /* ... */ }
```

Rebinding for a nested call shadows the outer value only inside that call — it never mutates it:

```java
ScopedValue.where(RequestContext.TENANT, otherTenant)
           .run(() -> migrationJob.copyFrom());   // outer binding intact afterwards
```

## Step 5 — Combine both: context that survives fan-out

This is where the two features pay off together. Scoped values are inherited by every subtask forked inside the scope, with no copying and no plumbing:

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

With `ThreadLocal` this only worked with `InheritableThreadLocal` plus a task-decorating executor — and silently broke whenever someone submitted work to a different pool.

## Step 6 — Migrating an existing pipeline

A safe, incremental order of operations:

1. **Inventory** your `ThreadLocal`s — find them and their `set` calls:

   ```bash
   grep -rn "ThreadLocal\|InheritableThreadLocal" src/main/java
   ```

2. **Classify** each one: *request-scoped and read-only after the edge* (→ `ScopedValue`), or *mutable accumulator* (→ pass an explicit object; scoped values cannot be reassigned).
3. **Bind at the edges only** — servlet filter, message listener, scheduled job entry point. One binding site per entry point, never in business code.
4. **Convert fan-outs**: replace `executor.submit(...)` + `Future.get()` clusters with a `StructuredTaskScope`, choosing the `Joiner` from Step 2.
5. **Delete the decorators**: `TaskDecorator`s, MDC-copying wrappers and `InheritableThreadLocal` hacks that existed only to move context across threads.
6. **Keep MDC bridged** if you log with SLF4J — logging frameworks still read the MDC (a `ThreadLocal`), so set it from the scoped value at the edge:

   ```java
   ScopedValue.where(RequestContext.TENANT, tenantId).run(() -> {
       MDC.put("tenant", tenantId.value());
       try { chain.doFilter(request, response); } finally { MDC.clear(); }
   });
   ```

7. **Enable preview** for the structured concurrency half only (`ScopedValue` needs nothing):

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

   Preview classes are also compiled with a minor-version marker, so tests and runtime must use the *same* JDK feature release. Split the migration if you cannot ship `--enable-preview` yet: adopt `ScopedValue` now, keep executors for fan-out.

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

## Common pitfalls (and how to detect them)

- **Leaking a `Subtask` outside the scope.** Calling `get()` after the `try` block, or storing the subtask in a field, defeats the whole model. Read results *inside* the block and return plain values.
- **Subtasks that ignore interrupts.** Cancellation is cooperative: a subtask blocked in a non-interruptible call delays `close()` indefinitely, because `close()` always waits for its threads. Make downstream clients interruptible and give them their own timeouts.
- **Using a scope from another thread.** `fork`, `join` and `close` may only be called by the owner thread; `join` may only be called once, and `fork` never after `join`. Do not stash a scope in a bean field — create it per call.
- **Reusing a `Joiner`.** A `Joiner` is stateful: create a fresh one per scope, never share or cache it.
- **Expecting `ScopedValue` to be mutable.** There is no `set()`. If code needs to *write* context, pass a mutable holder explicitly, or restructure to return values.
- **Unbound reads in background jobs.** `get()` on an unbound scoped value throws `NoSuchElementException`. Scheduled jobs and Kafka listeners are separate entry points and need their own binding — test that path explicitly:

  ```java
  assertThatThrownBy(() -> RequestContext.TENANT.get())
      .isInstanceOf(NoSuchElementException.class);
  ```

  Also note `orElse(null)` is rejected since Java 25 — use `isBound()` when absence is legitimate.
- **Assuming pinning is still the enemy.** Since Java 24 (JEP 491), blocking inside `synchronized` no longer pins a virtual thread, so the old `-Djdk.tracePinnedThreads` sweep is mostly obsolete. What remains are native frames and class-initialization blocking; watch the `jdk.VirtualThreadPinned` JFR event instead of the removed system property.
- **Backpressure still isn't free.** Structured fan-out makes concurrency easy to create; connection pools and downstream rate limits remain the real capacity ceiling. Cap deliberately with a `Semaphore` or a bounded pool.
- **Preview API drift.** Structured concurrency is on its fifth preview in Java 25 (a sixth is lined up for Java 26), so the API can still change between releases. Wrap fan-out in a thin internal helper (e.g. `Parallel.all(...)`) so an upgrade touches one class instead of hundreds of call sites.

## Outcome

The two features fix different halves of the same problem. `StructuredTaskScope` makes concurrency *lexically bounded*: failures cancel siblings, a configured timeout applies to the whole fan-out, and nothing outlives the method. `ScopedValue` — final as of Java 25 — makes context *immutable and bounded*, so request metadata flows into forked subtasks without decorators and without per-thread state that scales with a million virtual threads.

A checklist before adopting:

1. Move to Java 25 (LTS): `ScopedValue` ships unflagged and `synchronized` no longer pins carriers.
2. Migrate `ThreadLocal`s that are request-scoped and read-only to `ScopedValue`, bound only at entry points.
3. Replace `executor.submit` + `Future.get()` clusters with `StructuredTaskScope.open(...)` and an explicit `Joiner`.
4. Move per-call timeouts to a scope-wide `withTimeout`, and name scopes so thread dumps stay readable.
5. Bridge the logging MDC at the edge; delete inheritance hacks and task decorators.
6. Cover the unbound path (jobs, listeners) and the cancellation path with tests.
7. Isolate the still-preview structured concurrency API behind one helper class to keep upgrades cheap.

Virtual threads made concurrency cheap; structured concurrency and scoped values make it *safe*. Adopt them where a request fans out, and treat every entry point as a place where context must be bound explicitly.
