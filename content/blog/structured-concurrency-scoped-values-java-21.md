---
title: "Structured concurrency and scoped values in Java 21: replacing ThreadLocal in request pipelines"
date: 2026-08-22T00:00:00Z
description: "A hands-on tutorial on StructuredTaskScope and ScopedValue: fanning out calls safely and propagating request context without ThreadLocal."
---

## Problem

Once a service runs on virtual threads, two old habits start to hurt.

The first is **unstructured fan-out**. A request handler submits three downstream calls to an executor, collects the futures and waits. If one call fails, the others keep running: wasted work, leaked connections, and a latency floor set by the slowest sibling. Cancellation is manual, and error handling is a pile of `try/catch` around `Future.get()`.

The second is **`ThreadLocal` context propagation**. Tenant id, correlation id and user principal are traditionally stashed in a `ThreadLocal`. That works when threads are scarce and pooled — and breaks in two ways when they are not: the value is invisible to the child tasks you fan out to, and with millions of short-lived virtual threads each holding mutable per-thread state, memory and lifecycle become a real problem.

Java 21 addresses both: `StructuredTaskScope` (preview) gives concurrency a lexical scope, and `ScopedValue` (preview) gives context an immutable, bounded lifetime. This post is a step-by-step guide to both, with the migration path and the traps.

> Both APIs are **preview features** in Java 21 (`--enable-preview`). The concepts and shape below are what you design against today; the exact method names shifted in later JDKs — see the last section.

## Background: what "structured" buys you

Structured concurrency applies the rule that made structured programming work — *control flow enters and leaves a block at one place* — to threads:

```text
Unstructured (executor)                Structured (StructuredTaskScope)
-----------------------                --------------------------------
submit() escapes the method            forks live inside the try block
failure of one task is invisible       first failure can cancel siblings
cancellation is manual bookkeeping     scope close() joins everything
stack traces lose the caller           parent/child relation is explicit
```

The invariant: **when the block exits, no forked task is still running.** No orphans, no leaks.

## Step 1 — Fan out with `StructuredTaskScope`

The classic "fetch two things in parallel, fail fast" case:

```java
record Dashboard(User user, List<Order> orders) {}

Dashboard loadDashboard(long userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<User> user = scope.fork(() -> userClient.fetch(userId));
        Subtask<List<Order>> orders = scope.fork(() -> orderClient.fetchFor(userId));

        scope.join();            // wait for both (or for the first failure)
        scope.throwIfFailed();   // rethrow the first exception, wrapped

        return new Dashboard(user.get(), orders.get());
    }
}
```

What you get for free:

1. If `userClient.fetch` throws, the `orders` subtask is **interrupted immediately** — no wasted downstream call.
2. `scope.join()` returns only when every subtask is done, so nothing outlives the method.
3. Only call `Subtask::get` **after** a successful `join()`; before that it throws `IllegalStateException`.

Each `fork` runs on its own virtual thread, so a fan-out of 50 is as cheap as a fan-out of 2.

## Step 2 — Pick the right shutdown policy

Two policies ship with the API, and they cover most needs:

```java
// Fail fast: all results required (the "AND" case)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) { ... }

// Race: first success wins, losers are cancelled (the "OR" case)
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<Quote>()) {
    scope.fork(() -> providerA.quote(req));
    scope.fork(() -> providerB.quote(req));
    scope.join();
    return scope.result();   // fastest successful quote
}
```

Add a deadline for the whole fan-out instead of per-call timeouts:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var a = scope.fork(() -> serviceA.call());
    var b = scope.fork(() -> serviceB.call());

    scope.joinUntil(Instant.now().plusMillis(300));  // TimeoutException -> scope shuts down
    scope.throwIfFailed();
    return merge(a.get(), b.get());
}
```

Need "return what succeeded, ignore the rest"? Subclass and collect in `handleComplete`:

```java
class CollectSuccesses<T> extends StructuredTaskScope<T> {
    private final Queue<T> results = new ConcurrentLinkedQueue<>();

    @Override
    protected void handleComplete(Subtask<? extends T> subtask) {
        if (subtask.state() == Subtask.State.SUCCESS) {
            results.add(subtask.get());
        }
    }

    List<T> successes() {
        super.ensureOwnerAndJoined();
        return List.copyOf(results);
    }
}
```

`handleComplete` runs on the completing subtask's thread, so keep it short and use a concurrent collection.

## Step 3 — Replace `ThreadLocal` with `ScopedValue`

A `ScopedValue` is bound for the duration of a call, immutable inside it, and unbound automatically when the call returns:

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
TenantId tenant = RequestContext.TENANT.orElse(TenantId.SYSTEM);
if (RequestContext.TENANT.isBound()) { /* ... */ }
```

Rebinding for a nested call shadows the outer value only inside that call — it never mutates it:

```java
ScopedValue.where(RequestContext.TENANT, otherTenant)
           .run(() -> migrationJob.copyFrom());   // outer binding intact afterwards
```

## Step 4 — Combine both: context that survives fan-out

This is where the two features pay off together. Scoped values are inherited by every subtask forked inside the scope, with no copying and no plumbing:

```java
Report buildReport(TenantId tenantId) throws Exception {
    return ScopedValue.where(RequestContext.TENANT, tenantId).call(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var sales = scope.fork(this::loadSales);       // sees TENANT
            var costs = scope.fork(this::loadCosts);       // sees TENANT

            scope.join();
            scope.throwIfFailed();
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

## Step 5 — Migrating an existing pipeline

A safe, incremental order of operations:

1. **Inventory** your `ThreadLocal`s — find them and their `set` calls:

   ```bash
   grep -rn "ThreadLocal\|InheritableThreadLocal" src/main/java | grep -v "^.*Test"
   ```

2. **Classify** each one: *request-scoped and read-only after the edge* (→ `ScopedValue`), or *mutable accumulator* (→ pass an explicit object; scoped values cannot be reassigned).
3. **Bind at the edges only** — servlet filter, message listener, scheduled job entry point. One binding site per entry point, never in business code.
4. **Convert fan-outs**: replace `executor.submit(...)` + `Future.get()` clusters with a `StructuredTaskScope`, choosing the policy from Step 2.
5. **Delete the decorators**: `TaskDecorator`s, MDC-copying wrappers and `InheritableThreadLocal` hacks that existed only to move context across threads.
6. **Keep MDC bridged** if you log with SLF4J — logging frameworks still read the MDC (a `ThreadLocal`), so set it from the scoped value at the edge:

   ```java
   ScopedValue.where(RequestContext.TENANT, tenantId).run(() -> {
       MDC.put("tenant", tenantId.value());
       try { chain.doFilter(request, response); } finally { MDC.clear(); }
   });
   ```

7. **Enable preview** in build and runtime:

   ```xml
   <plugin>
     <artifactId>maven-compiler-plugin</artifactId>
     <configuration>
       <release>21</release>
       <compilerArgs><arg>--enable-preview</arg></compilerArgs>
     </configuration>
   </plugin>
   ```

   ```bash
   java --enable-preview -jar app.jar
   ```

## Common pitfalls (and how to detect them)

- **Leaking a `Subtask` outside the scope.** Calling `get()` after the `try` block, or storing the subtask in a field, defeats the whole model. Read results *inside* the block and return plain values.
- **Forgetting `throwIfFailed()`.** `join()` alone does not surface failures with `ShutdownOnFailure`; you will read a failed subtask and get `IllegalStateException` instead of the real cause.
- **Using a scope from another thread.** Scopes are confined to their owner thread; `fork`/`join` from elsewhere throws `WrongThreadException`. Do not stash a scope in a bean field — create it per call.
- **Expecting `ScopedValue` to be mutable.** There is no `set()`. If code needs to *write* context, pass a mutable holder explicitly, or restructure to return values.
- **Unbound reads in background jobs.** `get()` on an unbound scoped value throws `NoSuchElementException`. Scheduled jobs and Kafka listeners are separate entry points and need their own binding — test that path explicitly:

  ```java
  assertThatThrownBy(() -> RequestContext.TENANT.get())
      .isInstanceOf(NoSuchElementException.class);
  ```

- **Backpressure still isn't free.** Structured fan-out makes concurrency easy to create; connection pools and downstream rate limits remain the real capacity ceiling. Cap deliberately with a `Semaphore` or a bounded pool.
- **Preview API drift.** Compiling with `--enable-preview` means the API can change between JDK releases. Wrap fan-out in a thin internal helper (e.g. `Parallel.all(...)`) so an upgrade touches one class instead of hundreds of call sites.

## Outcome

The two features fix different halves of the same problem. `StructuredTaskScope` makes concurrency *lexically bounded*: failures cancel siblings, deadlines apply to the whole fan-out, and nothing outlives the method. `ScopedValue` makes context *immutable and bounded*, so request metadata flows into forked subtasks without decorators and without per-thread state that scales with a million virtual threads.

A checklist before adopting:

1. Confirm you are on Java 21+ and can ship `--enable-preview` (or plan the upgrade to the finalized API).
2. Replace `executor.submit` + `Future.get()` clusters with a scope and an explicit policy.
3. Add a whole-operation deadline via `joinUntil` and drop the per-call timeouts it replaces.
4. Move request-scoped, read-only `ThreadLocal`s to `ScopedValue`, bound only at entry points.
5. Bridge the logging MDC at the edge; delete inheritance hacks and task decorators.
6. Cover the unbound path (jobs, listeners) with tests.
7. Isolate the preview API behind one helper class to keep upgrades cheap.

Virtual threads made concurrency cheap; structured concurrency and scoped values make it *safe*. Adopt them where a request fans out, and treat every entry point as a place where context must be bound explicitly.
